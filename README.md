# paulovictorsantoselias

Este repositório contém projetos e experimentos de Paulo Victor Santos Elias.

## Estrutura

- Documentação de projetos pessoais
- Exemplos de código e estudos

## Como usar

Clone o repositório e explore os diretórios para exemplos e documentação.

```sh
git clone https://github.com/paulovictorsantoselias/<repo>
```

## Projeto hevik

```
quant-sec-project/
├─ README.md
├─ docker/
│  └─ Dockerfile
├─ requirements.txt
├─ qkd/                      # PoC BB84 (Qiskit)
│  ├─ __init__.py
│  ├─ bb84_sim.py
│  └─ demo_bb84.py
├─ pqc/                      # Post-quantum classical (liboqs)
│  ├─ __init__.py
│  ├─ kyber_demo.py
│  └─ key_hybrid.py
├─ app_integration/
│  ├─ server.py              # exemplo Flask que usa chaves híbridas
│  └─ encrypt_utils.py
├─ tests/
│  ├─ test_bb84.py
│  └─ test_kyber.py
└─ .github/
   └─ workflows/ci.yml
```

## Dependências

- qiskit>=0.40.0
- oqs==0.7.0          # se preferir liboqs-python / oqs (veja instruções)
- pycryptodome
- flask
- pytest

## Código Fonte

### qkd/bb84_sim.py

```python
import random
from qiskit import QuantumCircuit, Aer, execute
from qiskit.providers.aer import AerSimulator


def prepare_qubit(bit, basis):
    qc = QuantumCircuit(1, 1)
    # encode bit in Z ou X basis
    if bit == 1:
        qc.x(0)
    if basis == 'X':
        qc.h(0)
    qc.barrier()
    # measurement será feita pelo receptor
    return qc


def measure_qubit(qc, basis):
    q = qc.copy()
    if basis == 'X':
        q.h(0)
    q.measure_all()
    sim = AerSimulator()
    result = execute(q, sim, shots=1).result()
    counts = result.get_counts()
    measured_bit = int(list(counts.keys())[0][0])
    return measured_bit


def bb84_round(n_bits=64, eve=False, eve_prob=0.5):
    # Alice escolhe bits e bases
    alice_bits = [random.randint(0,1) for _ in range(n_bits)]
    alice_bases = [random.choice(['Z','X']) for _ in range(n_bits)]

    # Bob escolhe bases
    bob_bases = [random.choice(['Z','X']) for _ in range(n_bits)]

    alice_prepared = []
    for b, basis in zip(alice_bits, alice_bases):
        alice_prepared.append(prepare_qubit(b, basis))

    bob_results = []
    for i, qc in enumerate(alice_prepared):
        # possível eavesdropper
        if eve and random.random() < eve_prob:
            # Eve mede em uma base aleatória -> perturba o estado
            eve_basis = random.choice(['Z','X'])
            _ = measure_qubit(qc, eve_basis)  # colapso
            # após medição, re-prepara o bit colapsado na mesma base para encaminhar
            measured = _
            qc = prepare_qubit(measured, eve_basis)

        measured = measure_qubit(qc, bob_bases[i])
        bob_results.append(measured)

    # Sifting: mantém bits onde as bases coincidem
    key_indices = [i for i,(ab,bb) in enumerate(zip(alice_bases, bob_bases)) if ab==bb]
    alice_key = [alice_bits[i] for i in key_indices]
    bob_key = [bob_results[i] for i in key_indices]

    # Estima taxa de erro verificando uma amostra (aqui 10% da chave)
    sample_size = max(1, len(alice_key)//10)
    sample_idx = random.sample(range(len(alice_key)), sample_size) if len(alice_key)>0 else []
    mismatches = sum(1 for i in sample_idx if alice_key[i] != bob_key[i])
    error_rate = mismatches / sample_size if sample_size>0 else 0.0

    # Remove os bits da amostra da chave
    for idx in sorted(sample_idx, reverse=True):
        del alice_key[idx]
        del bob_key[idx]

    return {
        'raw_key_len': len(alice_bits),
        'sifted_key_len': len(alice_key),
        'alice_key': alice_key,
        'bob_key': bob_key,
        'error_rate': error_rate
    }


if __name__ == "__main__":
    res = bb84_round(n_bits=128, eve=True, eve_prob=0.3)
    print("Sifted len:", res['sifted_key_len'], "Error rate:", res['error_rate'])
```

### pqc/kyber_demo.py

```python
import oqs   # das bindings Python do liboqs
from Crypto.Cipher import AES
from Crypto.Random import get_random_bytes


def kyber_kem_demo():
    # Cria objeto KEM usando Kyber (assegure que o build o suporte)
    with oqs.KeyEncapsulation('Kyber768') as kem:
        public_key, secret_key = kem.generate_keypair()

        # Encapsula (Bob gera o ciphertext e shared secret)
        ciphertext, shared_secret_bob = kem.encapsulate(public_key)

        # Decapsula (Alice recupera o shared secret)
        shared_secret_alice = kem.decapsulate(ciphertext, secret_key)

        assert shared_secret_alice == shared_secret_bob
        return shared_secret_alice


def use_shared_secret_for_aes(shared_secret):
    # Usa os primeiros 32 bytes para chave AES-256 (exemplo)
    key = shared_secret[:32]
    iv = get_random_bytes(16)
    cipher = AES.new(key, AES.MODE_GCM, nonce=iv)
    msg = b"mensagem secreta de exemplo"
    ciphertext, tag = cipher.encrypt_and_digest(msg)
    return iv, ciphertext, tag


if __name__ == "__main__":
    ss = kyber_kem_demo()
    iv, c, t = use_shared_secret_for_aes(ss)
    print("encrypted len:", len(c))
```

### app_integration/server.py

```python
from flask import Flask, jsonify
from pqc.kyber_demo import kyber_kem_demo
from qkd.bb84_sim import bb84_round

app = Flask(__name__)

@app.route("/get_hybrid_key", methods=["GET"])
def get_hybrid_key():
    # PoC: gera shared secret do PQC e executa um round BB84 (simulado)
    pqc_ss = kyber_kem_demo()
    qkd_res = bb84_round(n_bits=256, eve=False)
    
    # Combine (XOR) partes dos dois para gerar a chave final (exemplo simples)
    qkd_key_bits = qkd_res['alice_key']
    # Converte bits para bytes
    qkd_key_bytes = bytes(
        int("".join(str(b) for b in qkd_key_bits[i:i+8]), 2)
        for i in range(0, len(qkd_key_bits) - len(qkd_key_bits) % 8, 8)
    )
    if not qkd_key_bytes:
        return jsonify({"error": "QKD key insufficient"})
    # Calcula quantas vezes repetir os bytes da chave QKD para igualar o tamanho do shared secret PQC
    qkd_repeat_count = (len(pqc_ss) // len(qkd_key_bytes)) + 1
    qkd_key_bytes_repeated = qkd_key_bytes * qkd_repeat_count
    # Faz o XOR entre cada byte do shared secret PQC e da chave QKD repetida
    combined = bytes(a ^ b for a, b in zip(pqc_ss, qkd_key_bytes_repeated))
    return jsonify({
        "combined_key_len": len(combined),
        "qkd_sifted_len": qkd_res['sifted_key_len'],
        "qkd_error_rate": qkd_res['error_rate']
    })


if __name__ == "__main__":
    app.run(port=5000)
```

### Dockerfile

```dockerfile
FROM python:3.11-slim

WORKDIR /app
COPY requirements.txt .
RUN apt-get update && \
    apt-get install -y build-essential libssl-dev libffi-dev git cmake && \
    git clone --branch main https://github.com/open-quantum-safe/liboqs.git && \
    cd liboqs && mkdir build && cd build && cmake .. && make && make install && \
    pip install --upgrade pip && pip install -r requirements.txt

COPY . /app
CMD ["python", "app_integration/server.py"]
```

### launch.json

```json
{
    "configurations": [
        {
            "type": "node",
            "request": "launch",
            "name": "Launch App",
            "program": "${workspaceFolder}/${input:programFile}"
        }
    ],
    "inputs": [
        {
            "id": "programFile",
            "type": "promptString",
            "description": "Enter the relative path to your main app file (for example, app.js, main.py, etc.)"
        }
    ]
}
```
# Encryption

## What it is

**Encryption** transforms plaintext into ciphertext using a key, making data unreadable without the corresponding decryption key. It protects **confidentiality** — even if an attacker intercepts or steals the data, they cannot read it without the key.

Two primary concerns:
- **Encryption in transit**: protect data as it moves over networks (TLS).
- **Encryption at rest**: protect data stored on disk, databases, backups.

---

## How it works

### Symmetric vs Asymmetric Encryption

| Dimension | Symmetric | Asymmetric |
|---|---|---|
| Keys | Single shared key | Key pair: public (encrypt) + private (decrypt) |
| Speed | Very fast (AES ~10 GB/s) | Slow (RSA ~100 KB/s) |
| Key distribution problem | Must share secret key securely | Public key can be shared freely |
| Use cases | Bulk data encryption | Key exchange, digital signatures, TLS handshake |
| Algorithms | AES-256-GCM, ChaCha20-Poly1305 | RSA-2048+, ECDH (P-256, X25519) |

### AES-256-GCM (Authenticated Encryption)

AES-GCM provides both **confidentiality** (encryption) and **integrity** (authentication tag). It detects tampering:

```python
from cryptography.hazmat.primitives.ciphers.aead import AESGCM
import os

# Encrypt
key = os.urandom(32)          # 256-bit key
nonce = os.urandom(12)        # 96-bit nonce — MUST be unique per encryption
plaintext = b"sensitive data"
aad = b"associated data"      # authenticated but not encrypted (e.g., metadata)

aesgcm = AESGCM(key)
ciphertext = aesgcm.encrypt(nonce, plaintext, aad)
# ciphertext includes 16-byte authentication tag at the end

# Decrypt
try:
    recovered = aesgcm.decrypt(nonce, ciphertext, aad)
    # Raises InvalidTag if ciphertext or aad was tampered with
except Exception:
    print("Decryption failed: data integrity compromised")
```

**Critical rule**: Never reuse a (key, nonce) pair with AES-GCM. Nonce reuse breaks both confidentiality and integrity. Use a random 96-bit nonce per encryption, or a counter (if carefully managed).

### RSA (Asymmetric)

RSA is used for encrypting small data (e.g., a symmetric key) and digital signatures:

```python
from cryptography.hazmat.primitives.asymmetric import rsa, padding
from cryptography.hazmat.primitives import hashes

# Key generation
private_key = rsa.generate_private_key(public_exponent=65537, key_size=2048)
public_key = private_key.public_key()

# Encrypt with public key (anyone can encrypt)
ciphertext = public_key.encrypt(
    b"secret-symmetric-key",
    padding.OAEP(mgf=padding.MGF1(algorithm=hashes.SHA256()), algorithm=hashes.SHA256(), label=None)
)

# Decrypt with private key (only the owner can decrypt)
plaintext = private_key.decrypt(ciphertext, padding.OAEP(...))
```

**Elliptic curve cryptography (ECC)** provides equivalent security to RSA with smaller keys:
- RSA-2048 ≈ ECC P-256 in security (but P-256 key is 32 bytes vs RSA's 256 bytes).
- Prefer ECDH (X25519) for key exchange and Ed25519 for signatures in new systems.

### TLS Handshake (Encryption in Transit)

TLS combines asymmetric crypto (for key exchange) with symmetric crypto (for bulk data transfer):

```
Client                                  Server
  |─── ClientHello ───────────────────► |
  |   TLS version, cipher suites,       |
  |   random_C                          |
  |                                     |
  |◄── ServerHello ─────────────────── |
  |    chosen cipher suite, random_S    |
  |◄── Certificate (server public key) ─|
  |◄── ServerKeyExchange (DH params) ── |
  |◄── ServerHelloDone ────────────── |
  |                                     |
  |  [Client verifies certificate       |
  |   against trusted CA chain]         |
  |                                     |
  |─── ClientKeyExchange ─────────────► |  (TLS 1.2: send pre-master secret)
  |  (TLS 1.3: DH key share)            |
  |                                     |
  |  Both sides derive session keys     |
  |  from master secret + randoms       |
  |                                     |
  |─── ChangeCipherSpec ──────────────► |
  |─── Finished (HMAC of handshake) ───► |
  |◄── ChangeCipherSpec ─────────────── |
  |◄── Finished ───────────────────── |
  |                                     |
  |═══════ Encrypted session (AES-GCM) ════════
```

**TLS 1.3 improvements over 1.2**:
- Removed weak cipher suites (RC4, 3DES, RSA key exchange without forward secrecy).
- 1-RTT handshake (vs 2-RTT in TLS 1.2).
- 0-RTT resumption for returning connections (with replay attack caveats).
- All cipher suites provide **forward secrecy** via ephemeral Diffie-Hellman.

```nginx
# Nginx TLS hardening
ssl_protocols TLSv1.2 TLSv1.3;
ssl_ciphers ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;
ssl_prefer_server_ciphers off;  # Let TLS 1.3 clients pick
ssl_session_timeout 1d;
ssl_session_cache shared:SSL:50m;
add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;
```

### Encryption at Rest

Three patterns depending on who manages the keys:

```
                ┌──────────────────────────────────────────────┐
                │           Encryption at Rest Models           │
                ├──────────────────┬───────────────────────────┤
                │ Application-Level│ Database-Level│ Infra-Level│
                │ Encryption (ALE) │ Encryption    │ (Disk/Cloud│
                ├──────────────────┼───────────────┼────────────┤
                │ App encrypts     │ DB engine      │ Cloud KMS  │
                │ before storing   │ encrypts       │ or disk    │
                │                  │ columns/rows   │ encryption │
                ├──────────────────┼───────────────┼────────────┤
                │ Granular control │ Column-level   │ Transparent│
                │ App owns keys    │ DB manages keys│ to the app │
                │ DB stores cipher │                │            │
                └──────────────────┴───────────────┴────────────┘
```

**Cloud KMS encryption at rest example**:
```python
# AWS KMS: Encrypt a database password before storing in config
import boto3

kms = boto3.client('kms')
key_id = 'arn:aws:kms:us-east-1:123456789:key/my-key-id'

# Encrypt
encrypted = kms.encrypt(KeyId=key_id, Plaintext=b'db-password-123')['CiphertextBlob']
# Store encrypted blob in config/secrets manager

# Decrypt at runtime
plaintext = kms.decrypt(CiphertextBlob=encrypted)['Plaintext']
```

### Envelope Encryption

Encrypting large data directly with KMS is slow and has size limits (4 KB max for AWS KMS). Use **envelope encryption**:

```
1. Generate a Data Encryption Key (DEK): random 256-bit key
2. Encrypt the data with the DEK (AES-256-GCM, fast)
3. Encrypt the DEK with the Key Encryption Key (KEK) in KMS
4. Store: [encrypted DEK | encrypted data]

To decrypt:
1. Send encrypted DEK to KMS → get plaintext DEK
2. Decrypt data locally using plaintext DEK
3. Zero out DEK from memory

Benefits:
- KMS only handles small key material (fast, audited)
- Bulk encryption done locally
- Different DEK per record = compromise of one DEK doesn't expose all records
```

```python
import os
from cryptography.hazmat.primitives.ciphers.aead import AESGCM

def encrypt_record(plaintext: bytes, kms_key_id: str) -> dict:
    # 1. Generate and encrypt DEK
    dek = os.urandom(32)
    encrypted_dek = kms.encrypt(KeyId=kms_key_id, Plaintext=dek)['CiphertextBlob']
    
    # 2. Encrypt data with DEK
    nonce = os.urandom(12)
    ciphertext = AESGCM(dek).encrypt(nonce, plaintext, None)
    
    # 3. Zero DEK (optional in Python due to GC, but conceptually important)
    
    return {"encrypted_dek": encrypted_dek, "nonce": nonce, "ciphertext": ciphertext}
```

### Key Management

| Tier | Description | Examples |
|---|---|---|
| **HSM** | Hardware Security Module — keys never leave hardware | AWS CloudHSM, YubiHSM, on-prem HSMs |
| **KMS** | Key Management Service — software-based, HSM-backed in cloud | AWS KMS, GCP Cloud KMS, Azure Key Vault |
| **Secrets Manager** | Store secrets (passwords, API keys), often wrapping KMS | AWS Secrets Manager, HashiCorp Vault |
| **App-level** | Application holds keys in memory, loaded from KMS/Vault | Custom via envelope encryption |

**Key rotation**: Periodically generate new KEKs. Re-encrypt all DEKs with the new KEK. Data doesn't need to be re-encrypted (only the DEK wrappers change).

### Hashing vs Encryption

| Property | Hashing (SHA-256, bcrypt) | Encryption (AES) |
|---|---|---|
| Reversible? | No (one-way) | Yes (with key) |
| Use cases | Passwords, data integrity, digital signatures | Data confidentiality |
| Key required? | No (bcrypt uses salt) | Yes |

---

## Key Trade-offs

| Choice | Pros | Cons |
|---|---|---|
| **Managed KMS** | Audited, FIPS 140-2, automatic rotation | Cloud vendor dependency, per-call cost |
| **Self-managed keys (HSM)** | Full control, compliance | Operational complexity, risk of losing keys |
| **Application-level encryption** | Column-level granularity, data protected from DB admins | App complexity, no SQL operations on encrypted data |
| **Transparent disk encryption** | Zero app changes | DB admin can read plaintext; no protection from application-layer attacks |
| **TLS 1.3 vs 1.2** | Forward secrecy, faster, removing weak ciphers | Compatibility with old clients |

---

## When to use

- **TLS everywhere**: All traffic in transit, internal and external.
- **Envelope encryption**: Whenever encrypting more than 4 KB of data with a cloud KMS.
- **Application-level encryption**: When columns contain PII/PHI that should be opaque even to database administrators (HIPAA, GDPR).
- **HSM**: Payment card data (PCI-DSS), government systems, certificate authority private keys.

---

## Common Pitfalls

- **Nonce reuse in AES-GCM**: reusing a (key, nonce) pair catastrophically breaks both confidentiality and forgery protection. Use a random 12-byte nonce generated per encryption.
- **Storing keys next to encrypted data**: if an attacker gets both the ciphertext and the encryption key, encryption provides zero protection. Keep keys in KMS/Vault, not in the same database or server.
- **Using ECB mode**: AES-ECB encrypts each 16-byte block independently, so identical plaintext blocks produce identical ciphertext blocks (the penguin problem). Always use GCM or CTR mode.
- **Trusting TLS certificate common name (CN)**: modern TLS validation checks Subject Alternative Names (SAN), not CN. Certificates without SANs are rejected by modern clients.
- **Disabling certificate verification in development**: `ssl_verify=False`, `InsecureSkipVerify: true` often gets committed to production config. Use self-signed certs with a dev CA instead.
- **Not rotating keys**: a key that has never been rotated is a liability. All KMS keys should have an annual rotation policy minimum.

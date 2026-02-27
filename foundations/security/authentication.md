# Authentication

## What it is

**Authentication** is the process of verifying **who you are** — confirming that a user, service, or device is who they claim to be. It is distinct from **authorization** (what you are allowed to do), though the two work together.

Authentication answers: "Are you really user-123?" Authorization answers: "Is user-123 allowed to access resource X?"

---

## How it works

### Password-Based Authentication

The baseline: user provides a username and password; server verifies.

**Password storage** — never store plaintext. Always hash with a modern slow hash:

```python
import bcrypt

# Registration: hash and store
password = "SecretP@ss123"
hashed = bcrypt.hashpw(password.encode('utf-8'), bcrypt.gensalt(rounds=12))
db.store_password(user_id, hashed)

# Login: verify
stored_hash = db.get_password(user_id)
is_valid = bcrypt.checkpw(password.encode('utf-8'), stored_hash)
```

**Why slow hashes (bcrypt, Argon2, scrypt)?**  
MD5 and SHA-256 are fast — an attacker with a leaked DB can try billions of passwords/second. Bcrypt with `rounds=12` takes ~250ms per check, reducing brute-force speed to thousands/second.

**Argon2** (winner of the Password Hashing Competition) is the current recommendation:

```python
from argon2 import PasswordHasher

ph = PasswordHasher(time_cost=2, memory_cost=65536, parallelism=2)
hash = ph.hash("SecretP@ss123")     # store this
ph.verify(hash, "SecretP@ss123")    # returns True or raises VerifyMismatchError
```

### Multi-Factor Authentication (MFA)

MFA requires two or more of:
- **Something you know**: password, PIN.
- **Something you have**: TOTP app (Google Authenticator), hardware key (YubiKey).
- **Something you are**: biometric (fingerprint, face ID).

**TOTP (Time-based One-Time Password):**
```python
import pyotp

# Setup: generate secret and show QR code to user
secret = pyotp.random_base32()
totp = pyotp.TOTP(secret)
qr_uri = totp.provisioning_uri(name="user@example.com", issuer_name="MyApp")

# Verification: user submits 6-digit code
totp.verify("123456")  # True if valid within 30s window (±1 window for clock drift)
```

### Session-Based Authentication

Traditional server-side sessions:

```
1. User submits login credentials
2. Server validates, creates session:
   session_id = random_string(32)
   session_store.set(session_id, {user_id: 123, expires: now+24h})
3. Server returns Set-Cookie: session_id=abc123; HttpOnly; Secure; SameSite=Lax
4. Client includes cookie on every subsequent request
5. Server looks up session_id in session store → identifies user

Logout: delete session from store → cookie is now invalid
```

**Session storage options**:
- In-memory (single server only — not horizontally scalable).
- Redis (fast, shared across all servers, TTL support).
- Database (durable but slower).

### JWT-Based Authentication (Stateless)

JSON Web Tokens encode the user's identity and claims in the token itself, signed by the server. No server-side session storage needed.

```
JWT Structure: header.payload.signature (base64-encoded, dot-separated)

Header:  { "alg": "HS256", "typ": "JWT" }
Payload: { "sub": "user-123", "email": "alice@example.com",
           "roles": ["user"], "iat": 1705000000, "exp": 1705003600 }
Signature: HMAC-SHA256(header + "." + payload, secret_key)
```

```python
import jwt
from datetime import datetime, timedelta

SECRET_KEY = "your-256-bit-secret"

# Issue token at login
token = jwt.encode({
    'sub': str(user.id),
    'email': user.email,
    'roles': user.roles,
    'iat': datetime.utcnow(),
    'exp': datetime.utcnow() + timedelta(hours=1)
}, SECRET_KEY, algorithm='HS256')

# Verify token on each request
try:
    payload = jwt.decode(token, SECRET_KEY, algorithms=['HS256'])
    user_id = payload['sub']
except jwt.ExpiredSignatureError:
    return 401, "Token expired"
except jwt.InvalidTokenError:
    return 401, "Invalid token"
```

**JWT advantages**: stateless — no session store needed; scales horizontally; can carry claims.  
**JWT disadvantages**: cannot revoke tokens before expiry without a blocklist; token size larger than session cookie.

**JWT revocation strategies**:
- Short-lived access tokens (15 min) + refresh tokens (stored server-side, can be revoked).
- Blocklist of revoked token JTIs (JWT IDs) in Redis — check on each request.

### API Key Authentication

For service-to-service and developer API access:

```python
# Issue: hash and store API key
import secrets, hashlib

raw_key = secrets.token_urlsafe(32)     # shown to user once
hashed_key = hashlib.sha256(raw_key.encode()).hexdigest()
db.store_api_key(user_id=123, hashed_key=hashed_key)

# Verify: hash the incoming key and compare
def verify_api_key(incoming_key: str) -> int | None:
    hashed = hashlib.sha256(incoming_key.encode()).hexdigest()
    record = db.lookup_api_key(hashed)
    return record.user_id if record else None
```

Store only the hash — if the database is breached, raw keys are not exposed.

### Authentication Flows

| Flow | Use Case |
|---|---|
| Username + Password + MFA | User login (web/mobile) |
| JWT (short-lived) + Refresh Token | API-authenticated SPAs and mobile apps |
| OAuth 2.0 / OIDC | Third-party login ("Login with Google") |
| API Key | Server-to-server, developer APIs |
| mTLS (mutual TLS) | Service-to-service in microservices |
| SAML | Enterprise SSO (legacy) |

---

## Key Trade-offs

| Approach | Pros | Cons |
|---|---|---|
| Sessions | Server-controlled, easy revocation | Requires shared session store (Redis) |
| JWT | Stateless, scales well | Can't revoke before expiry without blocklist |
| API Key | Simple for server-to-server | No expiry by default, must rotate on compromise |
| mTLS | Strongest service-to-service auth | Complex certificate management |

---

## When to use

- **Web/mobile user login**: username + password + MFA → JWT access token (15 min) + refresh token (30 days).
- **Third-party login**: OAuth 2.0 + OIDC → "Login with Google/GitHub."
- **Service-to-service**: mTLS (service mesh) or short-lived JWT signed by an internal identity service.
- **Developer APIs**: API keys with rate limiting and per-key scopes.

---

## Common Pitfalls

- **Storing passwords with fast hashes**: MD5, SHA-1, SHA-256 without salt are crackable in hours with rainbow tables. Use bcrypt, Argon2, or scrypt.
- **JWT signed with a weak secret or none (alg=none)**: a JWT with `alg=none` has no signature. Always verify the `alg` field and reject tokens with unexpected algorithms.
- **No MFA on admin/privileged accounts**: credential stuffing and phishing attacks make passwords insufficient for high-privilege accounts. Enforce MFA for all admin users.
- **Long-lived refresh tokens without rotation**: if a refresh token is stolen, the attacker has long-term access. Rotate refresh tokens on each use (single-use refresh tokens) and revoke on logout.
- **Cookies without Secure, HttpOnly, or SameSite**: missing `Secure` allows session theft over HTTP; missing `HttpOnly` enables XSS theft; missing `SameSite=Lax` enables CSRF attacks.

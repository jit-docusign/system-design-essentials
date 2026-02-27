# OAuth 2.0 and OpenID Connect

## What it is

**OAuth 2.0** is an authorization framework that lets a user grant a third-party application limited access to their resources on another service — without sharing passwords. It is a **delegation protocol**, not an authentication protocol.

**OpenID Connect (OIDC)** is an identity layer built on top of OAuth 2.0. It adds authentication by defining a standard way to verify who the user is and return their identity information in an **ID token**.

---

## How it works

### Core Roles

| Role | Description | Example |
|---|---|---|
| **Resource Owner** | End user with access to the protected resource | Alice |
| **Client** | Application requesting access on behalf of the user | A web app |
| **Authorization Server (AS)** | Issues tokens after authenticating the user | Auth0, Okta, Keycloak |
| **Resource Server (RS)** | Hosts the protected resources | Your API |

### OAuth 2.0 Flows

#### 1. Authorization Code + PKCE (Recommended for all clients)

PKCE (Proof Key for Code Exchange) closes a code interception vulnerability. Use this for web apps, SPAs, and mobile apps.

```
Browser          Client App       Authorization Server      Resource Server
   |                  |                   |                      |
   |─── Login ───────►|                   |                      |
   |                  |── Generate ───────|                      |
   |                  |   code_verifier   |                      |
   |                  |   code_challenge  |                      |
   |                  |                   |                      |
   |                  |─── GET /authorize?─────────────────────► |
   |                  |   response_type=code                     |
   |                  |   client_id                              |
   |                  |   redirect_uri                           |
   |                  |   code_challenge (SHA-256 of verifier)   |
   |                  |   scope=openid profile email             |
   |                  |                   |                      |
   |◄── Redirect to ──|──────────────────►|                      |
   |  Login page      |                   |                      |
   |── User logs in ──────────────────────►|                      |
   |                  |                   |                      |
   |◄─────────── Redirect back with code ─|                      |
   |── POST code ────►|                   |                      |
   |                  |─── POST /token ──►|                      |
   |                  |   code            |                      |
   |                  |   code_verifier ← matches challenge      |
   |                  |                   |                      |
   |                  |◄── access_token ──|                      |
   |                  |    refresh_token  |                      |
   |                  |    id_token       |                      |
   |                  |                   |                      |
   |                  |─── GET /api ──────────────────────────── ►|
   |                  |   Bearer access_token                    |
   |                  |                                          |
   |                  |◄────────────── Protected data ──────────|
```

```python
import hashlib, base64, secrets, httpx

# Step 1: Generate PKCE verifier + challenge
code_verifier = secrets.token_urlsafe(64)  # random 64-byte string
code_challenge = base64.urlsafe_b64encode(
    hashlib.sha256(code_verifier.encode()).digest()
).rstrip(b'=').decode()

# Step 2: Build authorization URL
auth_url = (
    "https://auth.example.com/authorize"
    f"?response_type=code"
    f"&client_id=my-app"
    f"&redirect_uri=https://myapp.com/callback"
    f"&scope=openid+profile+email"
    f"&code_challenge={code_challenge}"
    f"&code_challenge_method=S256"
    f"&state={secrets.token_urlsafe(16)}"  # CSRF protection
)

# Step 3: After redirect, exchange code for tokens
async def exchange_code(code: str) -> dict:
    resp = await httpx.AsyncClient().post(
        "https://auth.example.com/token",
        data={
            "grant_type": "authorization_code",
            "code": code,
            "redirect_uri": "https://myapp.com/callback",
            "client_id": "my-app",
            "code_verifier": code_verifier,  # proves we initiated the flow
        }
    )
    return resp.json()
    # Returns: {"access_token": "...", "refresh_token": "...", "id_token": "...", "expires_in": 3600}
```

#### 2. Client Credentials (Machine-to-Machine)

No user involved. A backend service authenticates itself to an API:

```python
async def get_service_token() -> str:
    resp = await httpx.AsyncClient().post(
        "https://auth.example.com/token",
        data={
            "grant_type": "client_credentials",
            "client_id": "service-a",
            "client_secret": "s3cr3t",
            "scope": "read:orders write:inventory",
        }
    )
    return resp.json()["access_token"]
```

#### 3. Device Authorization (TV / IoT / CLI)

For input-constrained devices. The device gets a code, user types it on another device:

```
curl -X POST https://auth.example.com/device/code \
  -d "client_id=my-cli&scope=openid"

# Response: {"device_code": "...", "user_code": "BCDF-4572", "verification_uri": "https://auth.example.com/activate"}

# Show user: "Go to https://auth.example.com/activate and enter BCDF-4572"

# Poll for token:
while true; do
  curl -X POST https://auth.example.com/token \
    -d "grant_type=urn:ietf:params:oauth:grant-type:device_code&device_code=..."
done
```

### OpenID Connect: ID Tokens

OIDC adds the `openid` scope. The authorization server returns an **ID token** (JWT) alongside the access token.

```json
// ID Token (JWT) decoded payload:
{
  "iss": "https://auth.example.com",       // Issuer
  "sub": "user-abc-123",                   // Subject (stable user ID)
  "aud": "my-app",                         // Audience (client_id)
  "exp": 1700000000,                       // Expiry
  "iat": 1699996400,                       // Issued at
  "nonce": "n-0S6_WzA2Mj",               // Replay protection
  "name": "Alice Smith",
  "email": "alice@example.com",
  "email_verified": true,
  "picture": "https://cdn.example.com/alice.jpg"
}
```

**Critical rule**: The ID token is a proof of authentication for the **client** only. Never pass an ID token to your backend API as an access credential. Use the access token for API calls.

### OIDC UserInfo Endpoint

After getting an access token with `scope=openid profile email`, call the UserInfo endpoint to get fresh user data:

```python
user_info = await httpx.AsyncClient().get(
    "https://auth.example.com/userinfo",
    headers={"Authorization": f"Bearer {access_token}"}
)
# Returns: {"sub": "user-123", "email": "alice@example.com", "name": "Alice Smith"}
```

### Token Types Comparison

| Token | Format | Contains | Lifetime | Usage |
|---|---|---|---|---|
| **Access Token** | JWT or opaque | Scopes, expiry, user claims | Short (15 min–1 hr) | Sent to Resource Server |
| **ID Token** | JWT only | User identity claims | Short (1 hr) | Consumed by Client only |
| **Refresh Token** | Opaque | None (reference) | Long (days–weeks) | Get new access token after expiry |

### Refresh Token Rotation

When a refresh token is used, issue a new refresh token and invalidate the old one:

```python
async def refresh_access_token(refresh_token: str) -> dict:
    resp = await httpx.AsyncClient().post(
        "https://auth.example.com/token",
        data={
            "grant_type": "refresh_token",
            "refresh_token": refresh_token,
            "client_id": "my-app",
        }
    )
    new_tokens = resp.json()
    # {access_token, refresh_token (NEW), expires_in}
    # Store new refresh_token, discard old one
    return new_tokens
```

If a stolen refresh token is used, the server detects the old (already-rotated) token being replayed and revokes the entire token family (the "refresh token rotation with reuse detection" pattern).

### OAuth vs SAML vs OIDC

| Dimension | OAuth 2.0 | SAML 2.0 | OIDC |
|---|---|---|---|
| Purpose | API authorization | SSO (enterprise) | Authentication + authorization |
| Token format | JWT / opaque | XML Assertions | JWT (ID Token) + OAuth tokens |
| Transport | HTTP redirects, REST | HTTP POST (XML) | HTTP redirects, REST |
| Age | 2012 | 2005 | 2014 |
| Mobile/SPA support | Excellent | Poor | Excellent |
| Enterprise SSO | Good | Excellent | Good |

---

## Key Trade-offs

| Choice | Pros | Cons |
|---|---|---|
| **Short access token + refresh** | Easy revocation, low server load | Extra round trip on expiry |
| **Long-lived access token** | Fewer requests | Can't revoke without token introspection |
| **Opaque access token** | Revocable instantly | Resource server must call AS to validate (introspection) |
| **JWT access token** | Self-contained, no AS call | Can't be revoked before expiry without blocklist |

---

## When to use

- Use **Authorization Code + PKCE** for any flow where users log in (web, SPA, mobile).
- Use **Client Credentials** for server-to-server API access with no user involvement.
- Use **OIDC** (add `openid` scope) whenever you need to identify the user (login/signup).
- Use **SAML** only when integrating with legacy enterprise identity providers that don't support OIDC.

---

## Common Pitfalls

- **Using implicit flow**: the old OAuth implicit flow returns tokens in URL fragments; it's deprecated. Use Authorization Code + PKCE instead.
- **Not validating the `state` parameter**: state binds the request to the callback, preventing CSRF attacks on the redirect. Always verify it matches.
- **Not validating the ID token**: always verify `iss`, `aud`, `exp`, and `nonce` before trusting an ID token. Library functions like `jwt.decode(token, options={algorithms:['RS256'], audience:'my-app'})` do this automatically.
- **Storing refresh tokens insecurely**: refresh tokens are long-lived credentials. Store them in HttpOnly cookies (for web) or in secure storage (for mobile), never in localStorage.
- **Scope creep**: requesting `scope=*` or all scopes by default violates least-privilege. Request only what the application needs.
- **Skipping refresh token rotation**: without rotation, a stolen refresh token is valid indefinitely. Rotate on every use and implement reuse detection.

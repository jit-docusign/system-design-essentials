# Stateless Services

## What it is

A **stateless service** is one where the server holds no per-user, per-request, or per-session state between requests. Every incoming request carries all the information needed to process it — or the service retrieves state from an external, shared store. Any instance of the service can handle any request with identical behavior.

Statelessness is the **foundational prerequisite** for horizontal scaling: if any instance can handle any request, you can add or remove instances freely without disrupting users.

---

## How it works

### Stateful vs Stateless — The Core Difference

**Stateful service (problem):**
```
Request 1: User logs in → instance stores session in local dict {session: user:42}
Request 2: Routed to different instance → no session found → user is "not logged in"
```

```python
# Stateful: in-process state stored per instance
sessions = {}  # local to each process — not shared

@app.route('/login', methods=['POST'])
def login():
    user = validate_credentials(request.form['email'], request.form['password'])
    token = generate_token()
    sessions[token] = user.id   # ← stored in this process only
    return jsonify({"token": token})

@app.route('/api/profile')
def profile():
    token = request.headers.get('Authorization')
    user_id = sessions.get(token)  # ← only works if same process handles request
    if not user_id:
        return 401
    return user_service.get_user(user_id)
```

**Stateless service (solution):**
```python
# Stateless: state stored in external shared store (Redis)

@app.route('/login', methods=['POST'])
def login():
    user = validate_credentials(request.form['email'], request.form['password'])
    token = generate_token()
    redis.setex(f"session:{token}", 3600, user.id)   # ← stored externally
    return jsonify({"token": token})

@app.route('/api/profile')
def profile():
    token = request.headers.get('Authorization')
    user_id = redis.get(f"session:{token}")   # ← any instance can look this up
    if not user_id:
        return 401
    return user_service.get_user(user_id)
```

Any instance can now handle either request because state lives in Redis, not the process.

### What Constitutes "State" That Must Be Externalized

| Type | Example | External store |
|---|---|---|
| Session / auth state | Logged-in user, session data | Redis |
| Shopping cart | Items in cart, quantities | Redis or DB |
| Long-running task progress | Job status, progress % | Redis or DB |
| In-progress uploads | Multi-part upload state | Object storage + DB |
| Rate limit counters | Per-user request counts | Redis |
| User preferences | Theme, locale | DB (query on each request) |

### JWT — Stateless Authentication

JSON Web Tokens (JWT) allow truly stateless authentication by moving session data into the token itself:

```
JWT structure: base64(header) + "." + base64(payload) + "." + signature

payload:
{
  "user_id": 42,
  "roles": ["user"],
  "exp": 1720003600,        ← expiry time
  "iss": "myapp.com"
}

Signature = HMAC(header + "." + payload, server_secret_key)
```

The server can verify any JWT by re-computing the signature — no external lookup needed:
```python
@app.route('/api/profile')
def profile():
    token = request.headers.get('Authorization').split(' ')[1]
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=['HS256'])
        user_id = payload['user_id']
    except jwt.InvalidTokenError:
        return 401
    return user_service.get_user(user_id)
```

**JWT trade-offs:**
- **Pros**: truly stateless — no Redis call; scales to any number of instances.
- **Cons**: cannot be revoked before expiry (the server can't "delete" a JWT); secrets must be rotated carefully; payload size increases each request.
- **Fix for revocation**: maintain a small token blacklist in Redis (only blocked tokens need to be stored, not all sessions).

### Designing Services for Statelessness

**Rule 1: No instance-local storage of user state**
- No in-process dicts, no local files, no instance-specific DB connections per user.

**Rule 2: Shared external state**
- Redis for ephemeral session data.
- Database for persistent user data.
- Object storage (S3) for files and blobs.

**Rule 3: Idempotent operations**
- If a request is retried (due to network timeout, LB retry), the second attempt should not cause double-effects.
- Use idempotency keys for state-changing operations:
  ```http
  POST /api/payments
  Idempotency-Key: "a3f7c9b1-4d2e-4f1a-8e3b-7f6d5c4b3a2e"
  ```
  Server stores the result by idempotency key; re-sends cached result on retry.

**Rule 4: Sticky sessions as a last resort only**
- Some legacy applications can't be made stateless quickly. A **sticky session** (or session affinity) routes all requests from a given user to the same instance. This allows the instance to maintain local state.
- Problem: sticky sessions defeat the purpose of horizontal scaling — a heavily-loaded instance can't shed its users. One instance crash means those users lose state.
- Use sticky sessions only as a migration step; the goal is full statelessness.

### Benefits of Stateless Services

```
Without statelessness:        With stateless services:
─────────────────────         ────────────────────────────────────────
One instance failing          Any instance can handle any request
  → that user's session lost  → load balancer transparently rereoutes
Scaling requires session       Scaling = add instances + register with LB
  migration complexity
Deployments require           Rolling deploys: drain old instances,
  session draining              new instances handle requests immediately
Load spikes require           Auto-scaling works correctly: spin up
  manual capacity planning       instances, scale down after spike
```

---

## Key Trade-offs

| Stateless | Stateful |
|---|---|
| Any instance handles any request | User must reach the same instance |
| Horizontal scaling trivial | Scaling requires session management |
| Resilient to instance failure | Instance failure = lost in-flight state |
| External state adds network latency | Local state is fast (no network hop) |
| Deployment/rolling updates simplified | Rolling updates require careful session draining |

---

## Common Pitfalls

- **Global variables and class-level state**: web frameworks reuse worker process memory across requests. A class-level variable (e.g., `class App: current_user = None`) is shared across all concurrent requests on that worker — severe concurrency bugs.
- **Local file writes for user content**: writing user-uploaded files to the local filesystem means other instances can't serve them. Write all user files to shared object storage (S3).
- **Sticky sessions hiding statefulness**: using sticky sessions allows launching today but creates scaling and availability debt. Invest in proper externalization.
- **JWT revocation gap**: if a user's token is stolen or account is compromised, a long-lived JWT can't be revoked without a blacklist — plan for token invalidation from day one.
- **Forgetting background workers**: a stateless web tier doesn't help if background job workers store job state locally. Externalize job state (task queue + DB) as well.

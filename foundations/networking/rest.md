# REST

## What it is

**REST (Representational State Transfer)** is an architectural style for designing networked APIs. It was defined by Roy Fielding in his 2000 doctoral dissertation as a set of constraints that, when followed, produce a scalable, uniform, stateless API over HTTP.

REST is the dominant API style for public and web-facing APIs. A system that follows all REST constraints is called **RESTful**.

---

## How it works

REST is defined by six architectural constraints:

### 1. Uniform Interface

The key constraint that distinguishes REST from other approaches. It has four sub-constraints:

- **Resource identification via URI**: every resource has a unique URL that identifies it.
  ```
  /users/42
  /orders/99/items/3
  ```
- **Manipulation through representations**: clients interact with resources through representations (JSON, XML), not through direct access to server state.
- **Self-descriptive messages**: each message contains enough information to describe how to process it (Content-Type header, HTTP method semantics).
- **HATEOAS** (Hypermedia As The Engine Of Application State): responses include links to related actions, allowing clients to navigate dynamically. Rarely implemented in practice but theoretically important.

### 2. Statelessness

Each request from a client must contain all the information the server needs to fulfill it. The server stores no client session state between requests.

```
✓ RESTful: Every request includes auth token + all context
  GET /users/42/orders?status=pending
  Authorization: Bearer <token>

✗ Not RESTful: "Remember that last time I asked you about user 42?"
  GET /my-pending-orders
  Session-ID: abc123  ← server maintains session state
```

**Benefit**: statelessness enables horizontal scaling — any server can handle any request.

### 3. Cacheable

Responses must declare whether they are cacheable. Cacheable responses can be stored by clients and intermediaries (CDNs, proxies) to reduce server load and improve performance.

```
Cache-Control: max-age=3600, public          ← can be cached 1 hour by anyone
Cache-Control: no-store                      ← must not be cached (private/sensitive)
ETag: "abc123"                               ← version identifier for conditional requests
```

### 4. Client-Server Separation

Client and server are separate concerns. The client handles UI; the server handles data storage. They evolve independently as long as the interface between them (the API) remains stable.

### 5. Layered System

A client doesn't know whether it's talking to a load balancer, cache, or the actual server. Intermediaries are transparent. This enables CDNs, API gateways, and caches without changing the client.

### 6. Code on Demand (Optional)

Servers can send executable code (JavaScript) to clients. The only optional constraint; rarely discussed.

---

## REST API Design Conventions

### Resources and URLs

Resources are **nouns**, not verbs. Actions are expressed via HTTP methods:

| Operation | HTTP Method | URL |
|---|---|---|
| List all users | GET | `/users` |
| Get one user | GET | `/users/42` |
| Create a user | POST | `/users` |
| Replace a user | PUT | `/users/42` |
| Update part of user | PATCH | `/users/42` |
| Delete a user | DELETE | `/users/42` |
| List user's orders | GET | `/users/42/orders` |

**Avoid verbs in URLs:**
- ✗ `/getUser?id=42`
- ✗ `/createOrder`
- ✓ `GET /users/42`
- ✓ `POST /orders`

### HTTP Status Codes in REST

| Status | Use |
|---|---|
| 200 OK | Successful read or update |
| 201 Created | Successful resource creation; include `Location` header pointing to new resource |
| 204 No Content | Successful delete or action with no response body |
| 400 Bad Request | Client sent invalid data |
| 401 Unauthorized | Authentication required or failed |
| 403 Forbidden | Authenticated but not authorized |
| 404 Not Found | Resource doesn't exist |
| 409 Conflict | State conflict (e.g., duplicate creation, version mismatch) |
| 422 Unprocessable Entity | Input is syntactically valid but semantically wrong |
| 429 Too Many Requests | Rate limit exceeded |
| 500 Internal Server Error | Generic server error |

### Versioning

REST APIs must be versioned to evolve without breaking existing clients:

```
URL versioning (most common):
  /v1/users
  /v2/users

Header versioning:
  Accept: application/vnd.myapi.v2+json

Query parameter versioning:
  /users?version=2
```

### Pagination

Large collections should be paginated:

```
Offset-based:
  GET /users?offset=100&limit=20
  → {"data": [...], "total": 500, "offset": 100, "limit": 20}

Cursor-based (preferred for large/dynamic datasets):
  GET /users?cursor=abc123&limit=20
  → {"data": [...], "next_cursor": "def456"}
```

---

## Key Trade-offs

| Trade-off | REST | GraphQL | gRPC |
|---|---|---|---|
| **Flexibility** | Fixed endpoints per resource | Client defines exact query | Fixed service methods |
| **Over/under-fetching** | Common | Solved | Solved |
| **Discoverability** | High (standard HTTP) | Medium | Low |
| **Browser support** | Native | Via HTTP | Needs proxy for browsers |
| **Performance** | Good | Good | Best (binary, streaming) |

---

## When to use

**REST is most appropriate when:**
- Building a public API consumed by external developers (standard, well-understood).
- Client diversity is high (web browsers, mobile apps, third-party integrations).
- Resource-oriented, CRUD-style operations align naturally with your domain model.
- Caching of responses is valuable (REST's cache semantics are well-defined; GraphQL's are harder).

---

## Common Pitfalls

- **Verbs in URLs** (`/getUser`, `/createOrder`): violates resource-oriented design; different from RPC-style APIs.
- **Always returning 200 OK with errors in the body**: breaks HTTP semantics; proper status codes make API health observable.
- **Over-fetching / under-fetching**: REST endpoints return fixed shapes. Clients may get too much data (wasted bandwidth) or need to make multiple requests (N+1 problem). Consider GraphQL for high data-flexibility requirements.
- **Ignoring idempotency**: PUT and DELETE are idempotent; POST is not. Design accordingly, especially for retry safety.
- **No versioning strategy**: unversioned APIs can't evolve without breaking clients. Always version from day one.
- **Using GET for state-changing operations**: GET requests are safe by HTTP convention — they may be cached, prefetched, or retried by proxies and browsers. Never use GET to trigger mutations.

# HTTP / HTTPS

## What it is

**HTTP (HyperText Transfer Protocol)** is the foundational application-layer protocol for data communication on the web. It defines how clients (browsers, mobile apps, services) send requests and how servers send responses. It is a **stateless, request-response** protocol.

**HTTPS** is HTTP secured with **TLS (Transport Layer Security)** — it encrypts the communication between client and server, authenticates the server's identity, and ensures data integrity. Virtually all production web traffic uses HTTPS.

---

## How it works

### Request-Response Model

Every HTTP interaction is initiated by the client:

```
Client                              Server
  │                                   │
  │── GET /users/42 HTTP/1.1 ────────▶│
  │   Host: api.example.com           │
  │   Accept: application/json        │
  │                                   │
  │◀── HTTP/1.1 200 OK ───────────────│
  │    Content-Type: application/json │
  │    {"id": 42, "name": "Alice"}    │
```

### HTTP Methods (Verbs)

| Method | Semantics | Idempotent | Safe |
|---|---|---|---|
| **GET** | Retrieve a resource | Yes | Yes |
| **POST** | Create a resource or trigger an action | No | No |
| **PUT** | Replace a resource entirely | Yes | No |
| **PATCH** | Partially update a resource | No (usually) | No |
| **DELETE** | Remove a resource | Yes | No |
| **HEAD** | Like GET but returns only headers | Yes | Yes |
| **OPTIONS** | Describe communication options | Yes | Yes |

- **Safe**: the operation does not modify server state.
- **Idempotent**: calling the same operation multiple times produces the same result.

### Status Codes

| Range | Category | Common Examples |
|---|---|---|
| 1xx | Informational | 100 Continue, 101 Switching Protocols |
| 2xx | Success | 200 OK, 201 Created, 204 No Content |
| 3xx | Redirect | 301 Moved Permanently, 302 Found, 304 Not Modified |
| 4xx | Client Error | 400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found, 429 Too Many Requests |
| 5xx | Server Error | 500 Internal Server Error, 502 Bad Gateway, 503 Service Unavailable, 504 Gateway Timeout |

### Key Headers

| Header | Direction | Purpose |
|---|---|---|
| `Host` | Request | Target server hostname |
| `Content-Type` | Both | Media type of the body (e.g., `application/json`) |
| `Accept` | Request | Media types the client can handle |
| `Authorization` | Request | Credentials (Bearer token, Basic auth) |
| `Cache-Control` | Both | Caching directives |
| `ETag` | Response | Version identifier for a resource |
| `Location` | Response | URL for redirects or newly created resources |
| `X-Request-ID` | Both | Distributed tracing correlation ID |

### HTTP Versions

| Version | Key Change |
|---|---|
| **HTTP/1.0** | New TCP connection per request |
| **HTTP/1.1** | Persistent connections (keep-alive), pipelining, chunked transfer encoding |
| **HTTP/2** | Binary framing, multiplexing (multiple requests over one TCP connection), header compression (HPACK), server push |
| **HTTP/3** | Runs over QUIC (UDP-based); eliminates TCP head-of-line blocking; faster connection setup |

**HTTP/2 multiplexing** is the key advancement for performance: in HTTP/1.1, browsers opened 6 parallel TCP connections to a host to work around head-of-line blocking; HTTP/2 handles multiple concurrent streams over a single connection.

### HTTPS / TLS

TLS adds three properties on top of HTTP:
1. **Encryption**: Data in transit is encrypted; cannot be read by intermediaries.
2. **Authentication**: The server presents a certificate signed by a trusted Certificate Authority (CA), proving its identity.
3. **Integrity**: Data cannot be tampered with in transit (MAC ensures integrity).

**TLS Handshake (simplified TLS 1.3):**
```
Client → Server: ClientHello (supported TLS versions, cipher suites, key share)
Server → Client: ServerHello (chosen cipher, key share, certificate)
Client validates certificate against trusted CAs
Client → Server: Finished (encrypted with derived key)
Server → Client: Finished
↓
Encrypted application data flows
```

TLS 1.3 reduced the handshake to **1 RTT** (from 2 RTT in TLS 1.2), and supports **0-RTT resumption** for reconnecting clients.

### Cookies and Sessions

HTTP is stateless — each request is independent. Cookies are the standard mechanism for maintaining state (sessions, authentication):

1. Server sets a `Set-Cookie: session_id=abc123; HttpOnly; Secure; SameSite=Strict` header.
2. Browser stores the cookie and sends it on subsequent requests with `Cookie: session_id=abc123`.
3. Server uses the cookie value to look up session state.

Modern applications increasingly use **JWTs (JSON Web Tokens)** as an alternative: the token is stored client-side (localStorage or cookie) and contains signed claims — no server-side session storage needed.

---

## Key Trade-offs

| Trade-off | Description |
|---|---|
| **Statelessness vs convenience** | Stateless = horizontally scalable; stateful sessions = easier to build but harder to scale |
| **Keep-alive vs connection costs** | Persistent connections reduce TLS/TCP setup overhead but consume server resources |
| **HTTP/2 vs HTTP/1.1** | HTTP/2 is better for high-concurrency web traffic; simpler clients may not support it |
| **HTTPS overhead** | TLS adds ~1 RTT and minor CPU overhead; the security benefit far outweighs the cost |

---

## When to use

- **HTTP GET** for idempotent reads; **POST** for actions that create or change state.
- **HTTPS everywhere**: there is never a good reason to use plain HTTP in production.
- **HTTP/2** for all web traffic where possible — use HTTP/3 (QUIC) for latency-sensitive mobile or global traffic.
- **HTTP streaming** (chunked transfer encoding, SSE) for long-running or incremental response scenarios.

---

## Common Pitfalls

- **Using GET for state-changing operations**: GET requests are cached and logged — using them for mutations can cause accidental data changes or security issues.
- **Returning 200 for errors**: APIs that return `200 OK` with `{"error": "not found"}` in the body break HTTP semantics and make error handling harder for clients. Use proper status codes.
- **Ignoring idempotency for retries**: If a POST creates a resource and the network fails before the response arrives, naive retry creates a duplicate. Use idempotency keys.
- **Not setting correct Cache-Control headers**: Browsers and CDNs cache responses aggressively if you don't explicitly control it. Private data must include `Cache-Control: no-store` or `private`.
- **Forgetting HSTS**: Switching from HTTP to HTTPS without HTTP Strict Transport Security (HSTS) allows downgrade attacks. Set `Strict-Transport-Security: max-age=31536000; includeSubDomains`.

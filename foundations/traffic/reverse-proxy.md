# Reverse Proxy

## What it is

A **reverse proxy** is a server that sits in front of one or more backend servers and intercepts requests from clients on their behalf. From the client's perspective, it is talking directly to the service — it has no visibility into the presence of the proxy or the backends behind it.

The term "reverse" distinguishes it from a **forward proxy** (which sits in front of clients and proxies their outgoing requests to the internet). A reverse proxy sits in front of **servers** and proxies **incoming** requests.

---

## How it works

```
Without reverse proxy:
  Client ──────────────────────────▶ Backend Server A
  Client ──────────────────────────▶ Backend Server B

With reverse proxy:
  Client ──────────────────────────▶ Reverse Proxy ──▶ Backend Server A
                                                    └──▶ Backend Server B
                                                    └──▶ Backend Server C
```

The reverse proxy:
1. Receives the client's request.
2. Optionally transforms it (add/remove headers, rewrite URL, validate auth).
3. Forwards it to one of the configured backends.
4. Receives the backend's response.
5. Optionally transforms the response (compress, cache, add security headers).
6. Returns the response to the client.

The backend addresses are never exposed to the client.

### Core Capabilities

#### TLS Termination

The reverse proxy terminates the TLS connection from the client. Traffic from the proxy to backends can be plain HTTP (within a trusted network), reducing the TLS overhead on every backend server.

```
Client ──(HTTPS/TLS)──▶ [Reverse Proxy — terminates TLS] ──(HTTP)──▶ Backend
```

Certificate management is centralized at the proxy. Backends don't need certificates.

#### Load Distribution

A reverse proxy inherently distributes load — it routes requests to the pool of backends using a configured load balancing algorithm.

#### Request Buffering

Slow clients (e.g., mobile on a poor connection) can upload a large request body slowly. The reverse proxy buffers the full request before forwarding it to the backend, freeing the backend from maintaining a slow connection open.

```
Mobile client (slow upload)  →  [Proxy buffers slowly]  →  Forwards complete request instantly to backend
```

This prevents "slowloris" attacks and protects backends from being tied up by slow clients.

#### Response Caching

The proxy can cache responses for idempotent requests (GET) and serve cached responses without forwarding to the backend:

```
GET /products/42 → Proxy checks cache → hit → returns cached response (no backend involved)
                                      → miss → forwards to backend, caches response, returns
```

#### Compression

Compress response bodies (gzip, brotli) before sending to the client. Offloads compression CPU from backends.

#### URL Rewriting and Path Routing

Forward requests to different backends based on URL path:

```
/api/* → API servers
/static/* → static file server
/admin/* → admin service
```

#### Header Manipulation

- **Add headers**: inject `X-Real-IP`, `X-Forwarded-For`, `X-Request-ID` for backend visibility.
- **Remove headers**: strip sensitive internal headers before returning responses to clients.
- **Security headers**: add `Strict-Transport-Security`, `X-Frame-Options`, `Content-Security-Policy` centrally.

#### Backend Anonymization / Security

The proxy hides the number, addresses, and technology of backend servers. Backends are not exposed to the public internet — they sit on a private network only reachable through the proxy.

### Reverse Proxy vs Load Balancer vs API Gateway

These concepts overlap significantly and are often implemented by the same software:

| | Reverse Proxy | Load Balancer | API Gateway |
|---|---|---|---|
| **Primary function** | Mediate client↔backend communication | Distribute traffic across backends | Entry point for microservices; policy enforcement |
| **Load balancing** | Often included | Core function | Often included |
| **TLS termination** | Yes | L7 LBs yes; L4 LBs no | Yes |
| **Auth enforcement** | Sometimes | No | Yes |
| **Rate limiting** | Sometimes | No | Yes |
| **Request transformation** | Yes | No | Yes |
| **Example** | Nginx, HAProxy, Envoy | AWS NLB/ALB | Kong, AWS API GW, Traefik |

In practice: a **load balancer** is a reverse proxy focused on traffic distribution. An **API gateway** is a reverse proxy with additional middleware. They are the same concept at different specialization levels.

---

## Key Trade-offs

| Trade-off | Description |
|---|---|
| **Single point of failure** | The reverse proxy is in the critical path; it must itself be highly available (run in HA pairs or use anycast) |
| **Latency** | Minimal additional hop compared to direct client→backend connection; generally sub-millisecond |
| **Operational complexity** | Centralizes cross-cutting concerns (TLS, headers, caching) but introduces another component to manage |

---

## When to use

A reverse proxy is used in almost every production web system. Specifically:
- **Always**: when exposing backend services to the internet — the proxy is the internet-facing entry point.
- **SSL termination**: when you don't want to manage certificates on every backend instance.
- **Static asset serving**: serve static files directly from the proxy (Nginx excels at this) without hitting application servers.
- **Microservices**: route different paths to different services.

---

## Common Pitfalls

- **Forwarding client IP incorrectly**: Backends see the proxy's IP, not the client's. Always set `X-Forwarded-For` or `X-Real-IP` headers, and have backends read from these headers (with proper validation to avoid spoofing).
- **Not handling the proxy SPOF**: A single reverse proxy instance is itself a single point of failure. Deploy multiple proxy instances with a floating VIP, DNS failover, or anycast.
- **Caching sensitive data**: Overly broad caching rules can cause responses containing personal data to be served to the wrong user. Carefully configure `Cache-Control` and respect `Vary` headers.
- **Ignoring upstream connection pooling**: The proxy should maintain a persistent connection pool to backends. Creating a new TCP connection per request wastes time and resources.

# Load Balancers (L4 vs L7)

## What it is

A **load balancer** distributes incoming network traffic across a pool of backend servers to:
1. **Prevent any single server from being overwhelmed**
2. **Improve availability**: if one server fails, traffic is routed to the healthy ones
3. **Enable horizontal scaling**: add more servers as demand grows

Load balancers operate at different layers of the network stack. The most important distinction is **Layer 4 (Transport)** vs **Layer 7 (Application)**.

---

## How it works

### Layer 4 (L4) Load Balancer — Transport Layer

L4 load balancers route traffic based solely on **network information**: source IP, destination IP, source port, destination port, and TCP/UDP protocol. They cannot inspect the contents of the packets.

```
Client: 10.0.0.1:54321 → LB → selects backend based on IP:port
LB routes: 10.0.0.1:54321 ↔ 10.10.0.3:8080  (backend 3)
```

**How routing decisions are made:**
- IP and port tuples
- TCP/UDP protocol
- Connection state (for TCP, the LB tracks which connections map to which backends via NAT tables)

**Modes of operation:**
- **NAT (Network Address Translation)**: LB rewrites source/destination IPs; all traffic passes through the LB.
- **Direct Server Return (DSR)**: Request goes through the LB; response goes directly from backend to client (bypassing the LB). Reduces LB bandwidth usage for response-heavy traffic.
- **IP tunneling**: Encapsulate client packets in new IP headers to route to distant backends.

**Characteristics:**
- Extremely fast — decisions made in microseconds with no content inspection.
- No awareness of HTTP, cookies, or sessions.
- Cannot route based on URL path, HTTP headers, or content.

**Examples**: HAProxy (TCP mode), hardware LBs (F5), Linux IPVS, cloud NLBs.

### Layer 7 (L7) Load Balancer — Application Layer

L7 load balancers **terminate the TCP connection** and **inspect the full HTTP request** before making a routing decision. They have visibility into everything: method, URL path, host header, query parameters, cookies, and body.

```
Client → L7 LB [reads: GET /api/users/42 Host: api.example.com Cookie: session=abc]
             ↓ routes based on URL path
         /api/* → API server pool
         /static/* → CDN or static server pool
         host: admin.example.com → admin server pool
```

**Routing capabilities:**
- URL-path-based routing (`/api` → service A, `/web` → service B)
- Host-header-based routing (virtual hosting: multiple domains on one LB)
- Cookie-based routing (sticky sessions: route same user to same backend)
- Header-based routing (route by `X-API-Version`, `Accept-Language`, etc.)
- Request modification: add/remove headers, rewrite URLs

**Additional features exclusively at L7:**
- **SSL/TLS termination**: decrypt HTTPS at the LB; backends receive plain HTTP (reduces per-backend TLS overhead)
- **Compression**: gzip/brotli compressing responses before they leave the LB
- **Caching**: cache responses for idempotent requests
- **Rate limiting**: enforce per-client/per-route limits
- **Authentication enforcement**: validate JWTs or API keys before requests reach backends
- **Health checks**: send real HTTP requests and validate response bodies (not just TCP connect)

**Examples**: Nginx, HAProxy (HTTP mode), Envoy, Traefik, cloud ALBs.

### L4 vs L7 Side by Side

| Property | L4 (Transport) | L7 (Application) |
|---|---|---|
| **Routing based on** | IP, port, TCP/UDP | URL, headers, cookies, content |
| **Protocol awareness** | No (raw TCP/UDP) | Yes (HTTP, gRPC, WebSocket) |
| **SSL termination** | No (pass-through only) | Yes |
| **Sticky sessions (cookies)** | No (IP hash only) | Yes (cookie-based) |
| **Health checks** | TCP connect | HTTP response code + body |
| **Performance** | Very high | High (slight overhead for inspection) |
| **Feature richness** | Low | High |

### Connection Handling

**L4 — Connection pass-through or NAT:**
The client and backend maintain a direct TCP connection (via NAT). The LB maintains a connection table mapping client→backend.

**L7 — Connection termination:**
The LB terminates the client's TCP+TLS connection. The LB then establishes a **separate TCP connection** to the backend (usually from a connection pool). This allows:
- TLS offloading
- Request buffering (protect backends from slow clients)
- Connection pooling (fewer backend connections)

### Health Checks

| Type | L4 | L7 |
|---|---|---|
| **TCP connect** | ✓ | ✓ |
| **HTTP GET /health → 200 OK** | ✗ | ✓ |
| **Response body validation** | ✗ | ✓ |
| **gRPC health check protocol** | ✗ | ✓ |

L7 health checks are far more representative of real service health — a server that accepts TCP connections but returns 500 errors on all requests is "down" at L7 but "up" at L4.

---

## Key Trade-offs

| | L4 | L7 |
|---|---|---|
| **Throughput** | Higher | Slightly lower |
| **Latency** | Slightly lower | Slightly higher |
| **Flexibility** | Limited | Very high |
| **Operational complexity** | Lower | Higher |
| **TLS offload** | Not possible | Built-in |

---

## When to use each

**L4 Load Balancer:**
- When you need maximum throughput and minimal latency (databases, internal TCP services)
- When content-based routing is not required
- As the first tier to handle raw connection distribution before passing to L7

**L7 Load Balancer:**
- For all HTTP/HTTPS traffic where content-based routing, SSL termination, or sticky sessions are needed
- As the primary internet-facing entry point for web services
- API gateways are L7 LBs with additional middleware (auth, rate limiting)

---

## Common Pitfalls

- **Using L4 for HTTP traffic where session affinity requires cookies**: L4 can only do IP hash for sticky sessions, which breaks when clients are behind NAT or share IPs. L7 cookie-based affinity is more reliable.
- **Not configuring health checks**: without proper health checks, the LB keeps sending traffic to unhealthy backends. Configure both L4 TCP checks and L7 HTTP checks where supported.
- **L7 LB as a bottleneck**: L7 LBs that terminate TLS and inspect all traffic can become throughput bottlenecks at very high scale. Use horizontal scaling of LBs or two-tier (L4 → L7) topologies.
- **Forgetting the LB itself as a SPOF**: a single LB instance is a single point of failure. Deploy LBs in HA pairs or use IP anycast for global resilience.

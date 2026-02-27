# API Gateway

## What it is

An **API Gateway** is the single entry point through which all client requests enter a microservices system. It sits at the edge between external clients (web browsers, mobile apps, third parties) and the internal cluster of backend services. Rather than clients calling individual services directly, they call the gateway, which handles cross-cutting concerns and routes requests to the appropriate upstream service.

The gateway is a specialised reverse proxy with an additional layer of **policy enforcement** and **protocol translation** baked in.

---

## How it works

```
Clients (Web, Mobile, Partners)
         │
         ▼
  ┌─────────────────────────────────────────┐
  │             API Gateway                  │
  │  ┌─────┐ ┌──────┐ ┌────────┐ ┌──────┐  │
  │  │ Auth│ │ Rate │ │Routing │ │ Obs- │  │
  │  │     │ │Limit │ │        │ │ erve │  │
  │  └─────┘ └──────┘ └────────┘ └──────┘  │
  └─────────────────────────────────────────┘
         │              │              │
  User Service    Order Service   Product Service
```

### Core Responsibilities

#### 1. Request Routing

The gateway maps incoming request paths/methods to the appropriate upstream service:

```
GET  /users/{id}           → User Service
POST /orders               → Order Service
GET  /products?category=X  → Product Service
GET  /search?q=shoes       → Search Service
```

Routing rules can be defined by: URL path prefix, host header, HTTP method, query parameter, or even request body content.

#### 2. Authentication & Authorization

The gateway validates credentials **before** any request reaches a backend service. This centralizes security enforcement — no individual service needs to implement auth logic:

- **JWT validation**: verify signature and claims; attach decoded user identity to forwarded request headers.
- **API key verification**: validate API key against a key store; attach associated account/scope context.
- **OAuth 2.0 token introspection**: call an auth service to validate opaque tokens.

If auth fails, the gateway rejects the request (401/403) and the backend never sees it.

#### 3. Rate Limiting & Throttling

Enforce per-client, per-endpoint request quotas to protect backends from overload and prevent API abuse:

```
Free tier users: 100 requests/minute
Paid tier users: 10,000 requests/minute
Unauthenticated: 10 requests/minute per IP
```

Rate limit counters are maintained in a shared store (Redis) for consistency across multiple gateway instances. See [Rate Limiting](rate-limiting.md) for algorithms.

#### 4. SSL/TLS Termination

Clients connect to the gateway over HTTPS. The gateway decrypts TLS, and forwards requests to backends over plain HTTP within the trusted internal network (or mTLS for zero-trust networks).

#### 5. Load Balancing

Route requests to a pool of instances of each upstream service, using a configured load balancing algorithm with health checks.

#### 6. Request / Response Transformation

- **Add headers**: inject auth context headers (`X-User-ID`, `X-Tenant-ID`) derived from the token.
- **Strip headers**: remove internal headers before sending responses to clients.
- **URL rewriting**: translate external-facing URLs to internal service URLs.
- **Protocol translation**: accept REST from client, translate to gRPC for internal service (or vice versa).

#### 7. Observability

- **Logging**: centralized access log for every request.
- **Metrics**: request count, error rate, latency per route — all visible at the gateway without touching individual services.
- **Distributed tracing**: generate and propagate a `X-Trace-ID` header through the entire request chain.

#### 8. Caching

Cache responses for read-heavy, stable endpoints (e.g., product catalog) to reduce upstream load and improve latency.

### Backend For Frontend (BFF) Pattern

In large systems, different client types (mobile app, web app, partner API) have different data needs. The **BFF pattern** creates dedicated gateways (or gateway layers) per client type:

```
Mobile App     → Mobile BFF (optimized for bandwidth, battery)
Web App        → Web BFF (richer data, server-side rendering support)
Partner API    → Partner Gateway (rate limits, auth, monetization)
         ↓              ↓                ↓
             Shared internal services
```

Each BFF can aggregate calls, shape responses, and apply policies tailored to that specific client.

### Gateway vs Service Mesh

| | API Gateway | Service Mesh |
|---|---|---|
| **Handles** | North-south traffic (client → cluster) | East-west traffic (service → service) |
| **Concerns** | Auth, rate limiting, routing, public API | mTLS, retries, circuit breakers, observability |
| **Placement** | Edge of the cluster | Sidecar on every service |
| **Examples** | Kong, AWS API GW, Traefik, Apigee | Istio, Linkerd, Envoy |

They are complementary, not competing: an API gateway handles the entry point; a service mesh handles internal communication.

---

## Key Trade-offs

| Trade-off | Description |
|---|---|
| **Centralized control vs coupling** | Centralizing auth and routing at the gateway is powerful but creates a dependency — the gateway must be updated as services evolve |
| **Performance** | Every request passes through; the gateway must be highly performant and horizontally scalable |
| **Single point of failure** | The gateway is in the critical traffic path; it must be deployed in HA with auto-scaling |
| **Monolithic gateway vs micro-gateways** | One central gateway is simple; per-team or per-domain gateways (BFF) give autonomy at the cost of duplication |

---

## When to use

- **Always** in a microservices architecture where external clients need to interact with multiple internal services.
- When you want to offload cross-cutting concerns (auth, rate limiting, SSL) from individual services.
- When building a public API product where monetization, versioning, and throttling are required.

---

## Common Pitfalls

- **Making the gateway a "God object"**: The gateway should enforce policies, not contain business logic. If the gateway knows about user objects, order state, or product pricing — those are business rules that belong in services.
- **Not scaling the gateway itself**: The gateway is a central choke point. It must scale horizontally as traffic grows; a fixed-size gateway becomes the bottleneck.
- **Synchronous auth on every request**: Calling an auth service for every request adds latency. Cache validated tokens at the gateway (with appropriate TTL).
- **Tight coupling between gateway config and service interfaces**: When a service changes its API, the gateway routing config must also change. Use service discovery and convention-based routing to reduce this coupling.

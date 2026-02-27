# Service Mesh

## What it is

A **service mesh** is a dedicated infrastructure layer that handles **service-to-service communication** in a microservices architecture. Rather than embedding networking logic (retries, load balancing, TLS, tracing) into each service, a service mesh externalizes it into a **sidecar proxy** (or host-level proxy) that intercepts all network traffic.

The result: services do plain HTTP/gRPC calls to each other; the mesh automatically handles reliability, security, and observability — transparently, without code changes.

Common implementations: **Istio** (Envoy-based), **Linkerd** (Rust-based, lightweight), **Consul Connect** (HashiCorp), **AWS App Mesh**.

---

## How it works

### Data Plane and Control Plane

```
                      Control Plane (Istiod / Linkerd Controller)
                      ┌──────────────────────────────────────────┐
                      │  - Certificate authority                  │
                      │  - Service registry + config distribution │
                      │  - Traffic policy enforcement             │
                      └────────────────────────────────────────┬─┘
                                                               │ xDS API (config push)
                                                               ▼
Service Pod A             Service Pod B             Service Pod C
┌────────────────┐        ┌────────────────┐        ┌────────────────┐
│ App Container  │        │ App Container  │        │ App Container  │
│                │        │                │        │                │
│ localhost:8080 │        │ localhost:8080 │        │ localhost:8080 │
└───────┬────────┘        └───────┬────────┘        └───────┬────────┘
        │ all traffic             │                         │
┌───────▼────────┐        ┌───────▼────────┐        ┌───────▼────────┐
│ Sidecar Proxy  │◄──────►│ Sidecar Proxy  │◄──────►│ Sidecar Proxy  │
│ (Envoy:15001)  │  mTLS  │ (Envoy:15001)  │  mTLS  │ (Envoy:15001)  │
└────────────────┘        └────────────────┘        └────────────────┘
```

**Data plane**: the sidecar proxies (Envoy) that intercept and route all network traffic.

**Control plane**: the centralized component (Istiod in Istio) that configures the data plane — pushes routing rules, TLS certificates, and policies to all proxies.

The application is unaware of the mesh — it talks to `localhost`, and iptables rules transparently redirect all traffic through the sidecar.

### Features Provided

**Traffic management:**
```yaml
# Istio VirtualService: canary routing — 90% to v1, 10% to v2
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: order-service
spec:
  http:
  - match:
    - headers:
        x-canary: {exact: "true"}
    route:
    - destination:
        host: order-service
        subset: v2
  - route:
    - destination: {host: order-service, subset: v1}
      weight: 90
    - destination: {host: order-service, subset: v2}
      weight: 10
```

**Mutual TLS (mTLS):** every connection between services is encrypted and authenticated automatically. No code changes. Certificates rotated by the mesh.

```yaml
# Enforce strict mTLS for the entire namespace
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT
```

**Retries and circuit breaking:**
```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: payment-service
spec:
  http:
  - retries:
      attempts: 3
      perTryTimeout: 2s
      retryOn: gateway-error,reset,retriable-4xx
    route:
    - destination:
        host: payment-service
```

**Observability (metrics, tracing, logs):** Envoy proxies automatically emit metrics (request rate, error rate, latency P50/P99) and generate distributed trace spans (Zipkin/Jaeger format) — no instrumentation code required.

### Service Mesh vs API Gateway

| Dimension | API Gateway | Service Mesh |
|---|---|---|
| Scope | North-south traffic (external → internal) | East-west traffic (service ↔ service) |
| Deployment | Single component at the edge | Sidecar per service instance |
| Common use | Auth, rate limiting, routing external requests | mTLS, retries, observability between services |
| Visibility | Single ingress point | Every service call |

They complement each other: the API gateway handles external traffic; the service mesh handles internal service-to-service traffic.

---

## Key Trade-offs

| Advantage | Disadvantage |
|---|---|
| Zero-code networking features | Significant operational complexity (running a mesh) |
| Consistent security policy across all services | Sidecar adds latency (~1ms per hop) and CPU/memory overhead |
| Strong security: mTLS everywhere by default | Steep learning curve; hard to debug mesh-level issues |
| Uniform observability without app changes | Adds infrastructure dependencies — mesh is a critical control plane |
| Enables canary, A/B, traffic shifting easily | Debugging: is the problem in my service or the proxy? |

---

## When to use

A service mesh is valuable when:
- You have **many microservices** (10+) and managing per-service TLS, retries, and observability is unsustainable.
- **Security compliance** requires encrypted and authenticated service-to-service communication.
- You need **progressive delivery** (canary, mirroring, A/B testing at the traffic level).
- You want **standardized observability** across services with different tech stacks.

Not worth the overhead when:
- You have a monolith or a handful of services.
- Your team lacks Kubernetes/mesh operational experience.
- Simple HTTP calls with client-side resilience libraries (Resilience4j, Polly) are sufficient.

---

## Common Pitfalls

- **Introducing the mesh before understanding it**: a misconfigured mesh silently drops traffic or causes authentication failures. Learn the basics (PeerAuthentication, VirtualService, DestinationRule) before deploying to production.
- **Treating the mesh as a silver bullet**: the mesh handles network faults, not application bugs. Business logic errors and database failures are still your responsibility.
- **Ignoring sidecar resource overhead**: Envoy sidecars consume ~50–150 MB RAM and real CPU. At 100 pods, that's 5–15 GB RAM just for proxies.
- **Debug complexity**: a request failure could be in the app, the sidecar, the control plane, or a policy rule. Use diagnostic tools (`istioctl analyze`, `istioctl proxy-config`) actively.
- **Not testing mTLS permissive → strict migration**: switching directly to strict mTLS without a permissive transition phase can break services that are not yet injected with sidecars.

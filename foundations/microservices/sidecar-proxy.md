# Sidecar Proxy

## What it is

The **sidecar proxy** is a design pattern in microservices where a **helper container or process** is deployed alongside the main service container, sharing its network namespace, and transparently handles cross-cutting networking concerns: TLS, retries, load balancing, observability, circuit breaking.

The "sidecar" name comes from the motorcycle sidecar — a separate attachment that rides alongside the main motorcycle (the service). The sidecar augments the vehicle without modifying the engine.

Most service mesh implementations (Istio, Linkerd, Consul Connect) use Envoy proxy as their sidecar.

---

## How it works

### Deployment Model

In Kubernetes, a pod can contain multiple containers. The sidecar proxy runs as a second container in the same pod, sharing the pod's network namespace.

```yaml
# Kubernetes Pod with sidecar (manual; normally injected by service mesh)
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: order-service          # main application container
    image: order-service:v2.1
    ports:
    - containerPort: 8080

  - name: envoy-proxy            # sidecar container
    image: envoyproxy/envoy:v1.28
    ports:
    - containerPort: 15001       # intercepts all inbound/outbound traffic
    volumeMounts:
    - name: envoy-config
      mountPath: /etc/envoy
```

**Automatic injection**: in a service mesh, you don't manually add the sidecar. The mesh injects it automatically via a Kubernetes **MutatingAdmissionWebhook** when the pod is scheduled.

```bash
# Label a namespace for automatic sidecar injection (Istio)
kubectl label namespace production istio-injection=enabled
```

### Traffic Interception

The sidecar uses **iptables rules** (init container sets them up) to transparently redirect all pod network traffic through itself:

```
Without sidecar:
  OrderService:8080 ──► network ──► PaymentService:8080

With sidecar (transparent):
  OrderService:8080
      │ (app thinks it's talking directly)
  iptables redirect all outbound → Envoy:15001
      │ Envoy forwards with TLS + retries + metrics
  network ──► Envoy:15001 (in PaymentService pod)
      │ iptables redirect inbound → Envoy
  PaymentService:8080
```

The application is completely unaware — it makes plain HTTP calls to the service name; the sidecar handles everything else.

### What the Sidecar Handles

| Concern | Description |
|---|---|
| **mTLS** | Certificates issued by the mesh CA; all connections encrypted and mutually authenticated |
| **Retries** | Configurable retry policies with per-try timeout and retry-on conditions |
| **Circuit breaking** | Open/close based on 5xx error rates or pending request count |
| **Load balancing** | Round-robin, least connection, consistent hash across endpoint instances |
| **Distributed tracing** | Generates and propagates trace spans (Zipkin/Jaeger) automatically |
| **Metrics** | Emits RED metrics (Rate, Error, Duration) for every request |
| **Rate limiting** | Per-source or global rate limits via Envoy's global rate limit service |
| **Traffic mirroring** | Shadow traffic to a new version for testing without affecting production |

### Envoy Configuration Example

```yaml
# Envoy upstream cluster with circuit breaking
static_resources:
  clusters:
  - name: payment_service
    connect_timeout: 0.5s
    type: STRICT_DNS
    load_assignment:
      cluster_name: payment_service
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address:
                address: payment-service
                port_value: 8080
    circuit_breakers:
      thresholds:
      - priority: DEFAULT
        max_connections: 100
        max_pending_requests: 50
        max_retries: 3
    outlier_detection:        # automatic unhealthy host ejection
      consecutive_5xx: 5
      interval: 10s
      base_ejection_time: 30s
```

### Ambassador and Adapter Variations

The sidecar pattern has two related variants:

**Ambassador**: a proxy that handles communication on behalf of the service with external systems — e.g., authenticating requests to a third-party API so the main service doesn't need credentials.

```
Service ──► Ambassador Sidecar ──► (adds auth headers) ──► External API
```

**Adapter**: translates the service's interface to a standard one expected by external consumers — e.g., converting a legacy service's custom log format to a structured JSON format readable by the logging platform.

```
Legacy Service (custom log format) ──► Adapter Sidecar ──► (converts to JSON) ──► Log Platform
```

---

## Key Trade-offs

| Advantage | Disadvantage |
|---|---|
| Cross-cutting concerns handled once, not per-service | Added latency ~0.5–1ms per request hop through proxy |
| Language-agnostic: works with Go, Java, Python, Ruby | Resource overhead: ~50–150 MB RAM per sidecar instance |
| Application code stays focused on business logic | Debugging requires understanding proxy config + app code |
| Mesh control plane can push config changes centrally | Additional dependency: sidecar crash (rare) affects the pod |
| Retroactively upgradeable: update proxy, not app | iptables-based interception adds complexity for low-level debugging |

---

## When to use

The sidecar proxy pattern is appropriate when:
- You're running **many microservices** and want to offload networking concerns uniformly.
- **Security compliance** requires authenticated, encrypted service-to-service communication without per-service TLS code.
- You need **consistent observability** across polyglot services (different teams, different languages).
- You want to implement **traffic management features** (canary, circuit breaking) without modifying application code.

Not worth the overhead for:
- Simple deployments with a handful of services.
- Teams without Kubernetes/service mesh familiarity.
- Latency-sensitive paths where every millisecond counts and 1ms proxy overhead is unacceptable.

---

## Common Pitfalls

- **Sidecar resource limits not set**: without CPU and memory limits, a misbehaving proxy can starve the application container. Always set resource requests/limits on the sidecar.
- **Debugging is harder**: when a request fails, is the problem in the application or the proxy? Learn to use `istioctl proxy-config`, `envoy admin API (/clusters, /listeners)`, and access logs actively.
- **Mesh not enabled for all services**: if some services have a sidecar and others don't, mTLS policies can require authentication that non-meshed services can't provide. Be consistent — either all services in a namespace are meshed, or configure permissive mode during migration.
- **Treating the sidecar as a replacement for application error handling**: the sidecar retries at the network level; it can't know if a request with a 200 status code was a business-level failure. Application-level validation is still required.

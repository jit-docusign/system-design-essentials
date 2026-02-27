# Service Discovery

## What it is

**Service discovery** is the mechanism by which microservices locate each other at runtime. When Service A needs to call Service B, it doesn't use a hardcoded IP and port — those change as containers are scheduled, restarted, and scaled. Instead, Service A queries a **service registry**, gets a healthy instance address, and makes the call.

Service discovery is the prerequisite for dynamic, scalable microservices — it enables horizontal scaling, rolling deployments, and health-based routing without manual configuration.

---

## How it works

### Two Modes

**Client-Side Discovery:** the client queries the registry directly and performs its own load balancing.

```
Client ──► Registry (Consul/Eureka) ──► returns list of [IP-1:8080, IP-2:8080, IP-3:8080]
Client applies load-balancing (round-robin, weighted) ──► calls IP-2:8080
```

Tools: Netflix Eureka + Ribbon, Consul with custom client, service registry library.

Pros: no extra hop; client controls load-balancing algorithm.  
Cons: each client must implement discovery logic; adds per-language coupling.

**Server-Side Discovery:** the client calls a load balancer or router; the load balancer queries the registry and forwards to an instance.

```
Client ──► Load Balancer ──► queries Registry ──► selects instance ──► forwards request
```

Tools: Kubernetes Services + kube-proxy, AWS ALB + ECS/EKS, Consul + Envoy.

Pros: client is simple; load balancer centralizes routing logic; language-agnostic.  
Cons: extra hop adds latency; the load balancer is a component to manage.

### Service Registry

The **service registry** is the authoritative directory of service instances. It stores:
- Service name
- Instance IP and port
- Health status
- Metadata (version, region, tags)

```
Registry contents:
  order-service:
    - 10.0.1.5:8080  [healthy, version=v2.1]
    - 10.0.1.6:8080  [healthy, version=v2.1]
    - 10.0.1.7:8080  [unhealthy] ← excluded from routing
  payment-service:
    - 10.0.2.1:8080  [healthy, version=v1.4]
```

**Registration**: services register on startup and deregister on graceful shutdown.

```python
# Consul registration on service startup
import consul

c = consul.Consul()
c.agent.service.register(
    name='order-service',
    service_id='order-service-' + INSTANCE_ID,
    address=HOST_IP,
    port=8080,
    check=consul.Check.http(f'http://{HOST_IP}:8080/health', interval='10s',
                             timeout='5s', deregister='30s')
)
```

**Health checks**: the registry periodically probes instances (HTTP, TCP, TTL). Unhealthy instances are removed from the routing pool.

```
Health check cycle (every 10s):
  Registry polls: GET http://10.0.1.7:8080/health
  Timeout after 5s → mark instance unhealthy
  After 30s → deregister and remove from DNS/routing
```

### DNS-Based Discovery (Kubernetes)

In Kubernetes, each **Service** object gets a stable DNS name. Kube-proxy programs iptables rules to route to healthy pods transparently.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: order-service
spec:
  selector:
    app: order-service      # routes to all pods with this label
  ports:
  - port: 8080
    targetPort: 8080
```

```python
# Calling order-service inside Kubernetes cluster:
# No IP, no port discovery — just use the DNS name
response = requests.post("http://order-service:8080/orders", json=order)
```

Kubernetes DNS resolution: `order-service.namespace.svc.cluster.local`

Pods come and go; the Service IP (ClusterIP) is stable. Kube-proxy updates iptables rules as pods are added/removed.

### Self-Registration vs Third-Party Registration

**Self-registration**: the service registers itself in the registry on startup.
- Simple; no external component needed.
- Problem: if the service crashes before deregistering, a stale entry remains until TTL expires.

**Third-party registration**: an external component (Kubernetes controller, AWS ECS agent, registrator) watches for new instances and registers/deregisters them.
- Decouples the service from registry knowledge.
- Requires the external component to reliably detect instance state.

### Load Balancing Algorithms at Discovery

When client-side discovery returns multiple instances, the client chooses via:

| Algorithm | Description |
|---|---|
| **Round-robin** | Rotate through instances in order |
| **Weighted round-robin** | Prefer instances with higher capacity |
| **Least connections** | Send to instance with fewest active calls |
| **Consistent hashing** | Route same client/key to same instance (session stickiness) |
| **Response-time weighted** | Prefer fastest-responding instances |

---

## Key Trade-offs

| Advantage | Disadvantage |
|---|---|
| Dynamic routing adapts to scale-out/in | Registry is a critical dependency — must be HA |
| Health-based routing avoids sending to dead instances | Stale registrations cause intermittent errors |
| Enables zero-downtime deployments (remove from registry before terminating) | Client-side: adds per-language library burden |
| Foundation for canary and blue-green routing | Server-side: extra network hop |

---

## When to use

Service discovery is necessary for any non-trivial microservices system:
- **Container orchestration** (Kubernetes, ECS): instances have ephemeral IPs — discovery is non-optional.
- **Horizontal auto-scaling**: new instances need to be discoverable immediately.
- **Blue-green and canary deployments**: control which version receives traffic.

For monolith-to-microservices migration, start with DNS-based discovery (Kubernetes Services) before adopting advanced client-side discovery.

---

## Common Pitfalls

- **Not implementing health checks**: without health checks, the registry serves stale entries for crashed instances. Every registered service must expose a `/health` endpoint and configure health checks.
- **TTL too long after deregistration**: if a crashed pod stays in the registry for 60+ seconds, that's 60 seconds of routing failures. Use aggressive deregistration TTLs (10–30s) with fast health check intervals.
- **Tight coupling to registry API**: if every service has direct Consul/Eureka SDK calls, switching registries is painful. Abstract discovery behind a standard interface or use platform-native discovery (Kubernetes Services).
- **Ignoring the registry in CI/CD**: during blue-green deployments, remove old instances from the registry before stopping them, and wait for health checks to confirm new instances before removing old ones.

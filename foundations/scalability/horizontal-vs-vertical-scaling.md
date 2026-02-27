# Horizontal vs Vertical Scaling

## What it is

**Horizontal scaling** (scaling out) adds more machines to a system to distribute load across multiple nodes. **Vertical scaling** (scaling up) increases the capacity of a single machine by adding more CPU, RAM, or storage.

Both approaches increase system capacity, but they differ fundamentally in their limits, costs, complexity, and application architecture requirements.

---

## How it works

### Vertical Scaling (Scale Up)

Upgrade individual servers to more powerful hardware:

```
Before:              After:
┌──────────┐         ┌──────────────────┐
│ 4-core   │   →     │  32-core         │
│ 16GB RAM │         │ 256GB RAM        │
│ 500GB SSD│         │  10TB SSD        │
└──────────┘         └──────────────────┘
```

**How it works**: cloud providers allow resizing instances without changing application architecture. Databases benefit most — more RAM means a larger buffer pool, reducing disk I/O.

**Limits**:
- Physics: there is a maximum machine size (currently ~448 vCPU, ~6TB RAM on cloud).
- Cost curve is not linear — largest instances cost disproportionately more per compute unit.
- Single point of failure: one large server is still one server.
- Downtime required (most cases): resizing usually requires stopping the instance.

**When vertical scaling is the right answer**: early in a system's lifecycle; databases (read replicas are complex, sharding is more complex); when horizontal scaling requires significant application changes.

### Horizontal Scaling (Scale Out)

Add more instances to a pool behind a load balancer:

```
Before:                     After:
                            ┌──────────┐
                     ┌─────►│ Server 1 │
                     │      └──────────┘
          ┌──────────┤      ┌──────────┐
Client───►│   LB     ├─────►│ Server 2 │
          └──────────┤      └──────────┘
                     │      ┌──────────┐
                     └─────►│ Server 3 │
                            └──────────┘
```

**How it works**: a load balancer distributes requests across N application instances. Each instance is identical and can serve any request.

**Requirements for horizontal scaling**:
- **Stateless services**: no session state stored in-process. If a user's second request hits a different server, it must work correctly. Move all state to an external store (Redis for sessions, DB for data).
- **Shared storage**: all instances read/write the same database, cache, and object store.
- **Idempotency**: operations that need to be retried must be safe to execute multiple times.

**Limits**:
- Does not help if the bottleneck is a non-horizontally-scalable component (single-writer database).
- Coordination overhead grows with instance count (distributed consensus, cache invalidation).
- Operational complexity: more servers to monitor, deploy, update.

### Scaling Comparison

| Dimension | Vertical | Horizontal |
|---|---|---|
| Implementation | No application changes needed | Requires stateless architecture |
| Single point of failure | Yes | No — N-1 redundancy |
| Upper bound | Machine size limit | Near-linear (with distributed bottlenecks) |
| Best fit | Databases, stateful services | Application / API tiers, microservices |
| Downtime for scaling | Usually (instance resize) | Zero (add instances live) |
| Cost | Super-linear at high end | More predictable, linear |
| Network overhead | None (single machine) | Inter-service calls add latency |

### Practical Scaling Order

Real systems typically apply a progression:

```
1. Vertical scale   → "just make the server bigger"
2. Read replicas    → scale reads horizontally while writes still go to one primary
3. Caching          → reduce DB load with Redis
4. Application horizontal scale → stateless app tier behind LB
5. DB connection pooling → handle more concurrent connections
6. Federation       → split DB by domain
7. Sharding         → horizontal write scaling for DB
```

Skipping steps wastes engineering effort. Many systems live comfortably at step 2–4 and never need step 5–7.

### The Stateless Prerequisite

The most critical requirement for horizontal scaling is **stateless services**:

```python
# Bad — state stored in process memory (won't work with multiple instances)
class App:
    session_store = {}  # in-process dict

    def login(self, user_id):
        session_id = generate_token()
        self.session_store[session_id] = user_id  # lost on load balancing to another instance
        return session_id

# Good — state stored in Redis (shared across all instances)
def login(user_id):
    session_id = generate_token()
    redis.setex(f"session:{session_id}", 3600, user_id)  # accessible from any instance
    return session_id
```

See [Stateless Services](stateless-services.md) for full coverage.

---

## Key Trade-offs

| Factor | Vertical | Horizontal |
|---|---|---|
| Simplicity | Simple — no architectural change | Requires stateless design + external state |
| Availability | Single server = single failure | Multiple servers = no single failure |
| Scalability ceiling | Hard upper bound | Soft ceiling (distributed coordination) |
| Cost at extreme scale | Expensive (big instances cost more per unit) | Cost-effective at large N |
| Operational overhead | Low | Higher (more instances, more monitoring) |

---

## When to use

**Vertical scaling first** when:
- The application is stateful and horizontal scaling requires significant refactoring.
- You're managing a relational database (horizontal DB scaling is complex).
- You need a quick fix with minimal risk.
- You're early in your product lifecycle and optimization time is scarce.

**Horizontal scaling** when:
- You've hit the vertical ceiling or the cost curve becomes unfavorable.
- You need high availability (no single point of failure).
- Your application tier is already stateless (or can be made stateless).
- You want to scale different components independently (e.g., scale API tier without scaling DB).

---

## Common Pitfalls

- **Horizontal scaling a stateful service**: distributing an application whose sessions live in process memory results in users getting logged out mid-session as requests move between instances. Make services stateless first.
- **Forgetting the bottleneck**: horizontally scaling the application tier when the database is the bottleneck changes nothing — all those extra app servers still queue on the same DB. Profile to find the actual bottleneck before scaling.
- **Not applying vertical scaling first**: rewriting a monolith to be stateless and distributable is a multi-week project. Vertical scaling may buy 6–12 months at a fraction of the cost.
- **Assuming linear scaling**: doubling instances rarely doubles throughput. Amdahl's Law limits parallel speedup when any serial bottlenecks exist (DB writes, shared locks, cross-service coordination). See [Performance vs Scalability](../core-concepts/performance-vs-scalability.md).

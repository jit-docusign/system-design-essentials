# Performance vs Scalability

## What it is

**Performance** measures how fast a system responds to a single request under a given load. **Scalability** measures how well a system maintains that level of performance as load increases.

A system can be:
- **Fast but not scalable** — responds quickly for one user, falls apart under 10,000 concurrent users.
- **Scalable but not fast** — handles millions of users, but every request takes 2 seconds.
- **Both fast and scalable** — the goal.

The distinction matters because the fixes are different. Performance problems are solved by optimizing the critical path. Scalability problems are solved by eliminating bottlenecks that worsen under load.

---

## How it works

### Performance

Performance is typically measured as **latency** — the time from when a request is sent to when a response is received. It is broken down at each layer:

```
Client → Network → Load Balancer → App Server → Database → App Server → Network → Client
  ↑                                                                                    ↑
  └────────────────────────────── Total Latency ──────────────────────────────────────┘
```

Key contributors to latency:
- **Compute** — CPU-intensive operations (serialization, encryption, business logic)
- **Memory** — cache misses that fall through to disk or remote calls
- **I/O** — disk reads/writes
- **Network** — round trips, bandwidth, DNS resolution
- **Synchronization** — locks, coordination between threads

Tools for measuring performance:
- **P50 / P95 / P99 latencies** — median and tail latencies; tail latency often reflects the worst user experience
- **Flame graphs** — visualize where CPU time is being spent
- **Profilers** — identify hot paths in code

### Scalability

Scalability describes the relationship between load and performance. A system **scales linearly** if doubling the resources doubles the capacity. In practice, overheads like coordination, synchronization, and network communication make this hard.

**Amdahl's Law** sets a fundamental limit on parallelization:

$$S(n) = \frac{1}{(1 - p) + \frac{p}{n}}$$

Where:
- $S(n)$ = speedup with $n$ workers
- $p$ = fraction of the task that is parallelizable

This means if 10% of your system is serial (not parallelizable), the maximum theoretical speedup is 10x, no matter how many machines you add.

**Universal Scalability Law (USL)** extends Amdahl further. It adds a second penalty term for **coherence overhead** — the cost of keeping distributed state consistent as the number of nodes grows:

$$C(n) = \frac{n}{1 + \alpha(n-1) + \beta n(n-1)}$$

Where:
- $n$ = number of processors/nodes
- $\alpha$ = contention penalty (serialisation — same as Amdahl's serial fraction)
- $\beta$ = coherency penalty (the cost of nodes coordinating with each other, which grows as $n^2$)

The critical insight: when $\beta > 0$, **throughput peaks at some finite $n$ and then decreases as you add more nodes**. This is why adding shards, replicas, or Kafka partitions beyond a certain point makes things worse, not better. It explains empirically observed behaviour that Amdahl alone cannot: distributed systems that get slower as you scale them. And it gives you a concrete diagnostic: if adding nodes is not improving throughput, first check whether $\alpha$ (lock contention, serial bottlenecks) or $\beta$ (cross-node coordination, gossip overhead, metadata synchronisation) is the dominant term.

---

## Key Trade-offs

| Trade-off | Description |
|---|---|
| Optimization vs simplicity | Micro-optimizing code for performance often increases complexity and reduces maintainability |
| Caching (perf) vs freshness | Caching dramatically improves performance but can serve stale data |
| Synchronous (fast locally) vs asynchronous (scales better) | Sync calls are simpler; async decoupling scales better under load |
| Premature optimization | Tuning performance before you know where the bottleneck is wastes effort |

---

## When to use / apply each lens

**Focus on performance when:**
- A single operation (query, API call, render) is too slow
- Latency SLOs are being breached at low traffic
- The critical path has obvious inefficiencies (N+1 queries, no caching)

**Focus on scalability when:**
- The system is fast at low concurrency but degrades as users increase
- A single node is becoming a bottleneck
- You expect 10x–100x traffic growth

**Distinguishing them diagnostically**: Run the system at 1 concurrent user and measure latency. Now run it at 100 concurrent users — if latency is similar, you have a scalability budget to spend. If latency at 1 user is already bad, you have a performance problem. If latency at 100 users is worse than at 1 user by more than the queueing theory prediction (M/M/1), you have a scalability problem masked as a latency problem — usually a serial bottleneck or a resource that does not parallelise.

**Load shedding vs backpressure**: When a system approaches capacity, it must protect itself. Two strategies:

| Strategy | Mechanism | Effect |
|---|---|---|
| **Backpressure** | Slow down or block producers when the consumer queue is full | Propagates the capacity constraint upstream; the source generates less work | 
| **Load shedding** | Actively reject excess requests (return 503) when above a threshold | Protects the service by killing work already in the system; callers must retry or degrade | 

Backpressure is preferable in internal systems where callers can throttle. Load shedding is necessary at the edge where callers (users, external services) cannot throttle — and it is the only way to prevent a slow downstream dependency from causing a request queue to grow unbounded and consume all memory. A system with no load shedding will eventually OOM under overload.

---

## Common Pitfalls

- **Confusing the two**: "Our system is slow" — is it slow for one user, or only under load? The diagnosis differs entirely.
- **Ignoring tail latency (P99)**: Optimizing for P50 while P99 is 10x higher means a large percentage of users have a poor experience.
- **Scaling before profiling**: Adding machines won't help if the bottleneck is a single-threaded process, a global lock, or a sequential database query.
- **Over-engineering for scale that never comes**: The cost of building a massively scalable system upfront often isn't justified. Start simple, measure, then scale what the data tells you to.
- **Ignoring the data layer**: Application servers are easy to scale horizontally. Databases are not. Most scalability problems eventually land at the database.
- **Not knowing which USL term dominates**: When adding nodes fails to improve throughput, the cause is either contention ($\alpha$ — serial bottleneck) or coherence overhead ($\beta$ — cross-node coordination). These have different fixes. $\alpha$ is fixed by removing serial code paths and locks. $\beta$ is fixed by reducing coordination surface area: fewer partitions that must coordinate, more local state, lower-frequency gossip.
- **No load shedding at the edge**: A service with no mechanism to reject excess load will pile up request queues until it OOMs. Circuit breakers and rate limiting protect downstream; explicit 503 shedding protects the service itself.

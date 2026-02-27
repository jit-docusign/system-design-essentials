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

**Universal Scalability Law (USL)** extends this further, accounting for the coordination overhead that grows as parallelism increases.

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

---

## Common Pitfalls

- **Confusing the two**: "Our system is slow" — is it slow for one user, or only under load? The diagnosis differs entirely.
- **Ignoring tail latency (P99)**: Optimizing for P50 while P99 is 10x higher means a large percentage of users have a poor experience.
- **Scaling before profiling**: Adding machines won't help if the bottleneck is a single-threaded process, a global lock, or a sequential database query.
- **Over-engineering for scale that never comes**: The cost of building a massively scalable system upfront often isn't justified. Start simple, measure, then scale what the data tells you to.
- **Ignoring the data layer**: Application servers are easy to scale horizontally. Databases are not. Most scalability problems eventually land at the database.

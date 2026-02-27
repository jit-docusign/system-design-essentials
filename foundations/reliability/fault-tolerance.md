# Fault Tolerance

## What it is

**Fault tolerance** is the ability of a system to continue operating correctly — or degrade gracefully — when one or more of its components fail. A fault-tolerant system does not simply mask failures: it anticipates them, isolates their impact, and keeps the system serving users at an acceptable level of functionality.

The key insight is that in distributed systems, **failure is not an exception — it is the norm**. Hardware fails, networks partition, software crashes, and third-party services degrade. A fault-tolerant design assumes this and plans for it.

---

## How it works

### Types of Failures

| Type | Description | Example |
|---|---|---|
| **Crash failure** | A node stops responding entirely | Server OOM kill, hardware failure |
| **Omission failure** | A node fails to send or receive messages | Dropped packets, network partition |
| **Timing failure** | A node responds too slowly | GC pause causing timeout, disk I/O spike |
| **Byzantine failure** | A node responds with incorrect/malicious data | Bit-flip corruption, buggy third-party service |

Byzantine failures are the hardest to handle and rare in internal systems. Most distributed systems design focuses on crash and timing failures.

### Key Patterns

#### Timeouts

Every network call should have a timeout. Without one, a single slow downstream service can hold a thread indefinitely, eventually exhausting your thread pool and causing a cascading failure.

```
No timeout:
  Service A → Service B (hung) → Thread held indefinitely
  After N requests, all threads are waiting → Service A becomes unresponsive

With timeout (e.g., 500ms):
  Service A → Service B (500ms) → Timeout → Error returned immediately
  Thread freed; Service A stays healthy
```

**Guideline**: Set timeouts based on SLO percentiles. If your P99 latency to a dependency is 200ms, set a timeout of 500ms–1000ms, not 30s.

#### Retries with Exponential Backoff

Transient failures (brief network hiccups, brief overload) can often be resolved by retrying. But naive immediate retries can amplify load on an already-struggling service.

**Exponential backoff + jitter:**
```
Attempt 1 → fail → wait 100ms
Attempt 2 → fail → wait 200ms + jitter
Attempt 3 → fail → wait 400ms + jitter
Attempt 4 → fail → wait 800ms + jitter
```

**Jitter** (random offset) prevents the "thundering herd" problem where all retrying clients retry at exactly the same time.

**Critical**: Only retry **idempotent** operations. Retrying a non-idempotent operation (e.g., charging a credit card) can cause duplicate actions.

#### Circuit Breaker

A circuit breaker sits in front of a downstream call and tracks its failure rate. If failures exceed a threshold, it "opens" the circuit — subsequent calls fail immediately without attempting the network call.

States:
- **Closed**: Normal operation; requests flow through.
- **Open**: Too many failures; requests fail fast without calling the downstream.
- **Half-Open**: After a timeout, a few probe requests are sent; if they succeed, circuit closes.

```
[Closed] → failure rate exceeds threshold → [Open]
[Open]  → probe timeout elapsed → [Half-Open]
[Half-Open] → probes succeed → [Closed]
[Half-Open] → probes fail → [Open]
```

Benefits: prevents cascading failures, reduces load on a struggling downstream, provides a fast failure path for clients.

#### Bulkheads

Borrowed from ship design — in a ship, watertight compartments prevent a single hull breach from sinking the vessel.

In software: **isolate resources (thread pools, connection pools, queues) between dependencies** so that overloading one does not starve others.

```
Without bulkheads:
  Shared thread pool → Service B goes slow → all threads blocked → Service A and C also starved

With bulkheads:
  Pool for Service B (10 threads) → Service B goes slow → only 10 threads blocked
  Pool for Service A (20 threads) → unaffected
  Pool for Service C (20 threads) → unaffected
```

#### Graceful Degradation

When a component fails, return a degraded but useful response rather than a complete failure. Examples:
- **Recommendation service down**: show popular items instead of personalized recommendations.
- **Cache unavailable**: fall through to the database with a warning.
- **Search service down**: disable search; keep the rest of the product functional.

Design every feature with a degraded fallback: "If X is unavailable, what is the best experience we can offer?"

#### Idempotency

Make operations idempotent — calling them multiple times produces the same result as calling them once. This is essential for safe retries.

Techniques:
- Include an **idempotency key** in requests (a unique ID for the operation).
- The server checks if the key has already been processed; if so, return the cached result.
- Used extensively in payment systems and message processing.

---

## Key Trade-offs

| Pattern | Benefit | Cost |
|---|---|---|
| Timeouts | Prevents thread exhaustion | Adds false failures on slow-but-valid responses |
| Retries | Recovers from transient failures | Amplifies load; risks duplicate actions on non-idempotent ops |
| Circuit breakers | Prevents cascading failure | Adds complexity; requires tuning thresholds |
| Bulkheads | Isolates failure blast radius | Higher resource consumption; more configuration |
| Graceful degradation | Keeps system partially usable | Requires designing and maintaining fallback paths |

---

## When to apply

- **Always use timeouts** on any I/O, network, or external call.
- **Always retry with backoff + jitter** for transient errors on idempotent operations.
- **Add circuit breakers** to calls to any services that are not under your direct control or that have historically been unreliable.
- **Use bulkheads** when different client types or downstream dependencies should not be able to starve each other (e.g., isolate the thread pool for background jobs from user-facing requests).
- **Design graceful degradation** for every non-critical feature your system depends on.

---

## Common Pitfalls

- **No timeouts**: The single most common cause of cascading failures. Every external call must have a timeout.
- **Retrying non-idempotent operations**: Retrying a payment charge can double-charge the user. Always ensure idempotency before adding retries.
- **Overly aggressive circuit breaker thresholds**: A circuit breaker that opens on a single failure will cause unnecessary outages. Tune thresholds based on observed baseline error rates.
- **Ignoring the "half-open" state**: Systems often handle closed and open states correctly but forget to implement or test the half-open recovery path.
- **Treating all errors as retriable**: Errors caused by bad input (4xx HTTP errors) should not be retried — they will always fail. Only retry transient errors (5xx, network timeouts).

# Circuit Breaker

## What it is

A **circuit breaker** is a resilience pattern that prevents repeated calls to a failing downstream service. It detects when a dependency is unhealthy and **short-circuits** subsequent calls — failing fast instead of waiting for timeouts — until the dependency recovers.

The name comes from electrical circuit breakers: when an electrical fault is detected, the breaker trips (opens) to prevent further damage to the circuit. Once the fault is resolved, the breaker can be reset.

Without circuit breakers, a failing downstream service cascades into upstream service failures: threads are blocked waiting for timeouts, connection pools exhaust, and the entire system degrades even though only one dependency is unhealthy.

---

## How it works

### Three States

```
┌──────────────────────────────────────────────────────────┐
│                                                          │
│    [CLOSED] ──── error rate > threshold ──▶ [OPEN]      │
│       ↑                                        │         │
│       │                                   probe timeout  │
│       │                                        │         │
│   probes succeed                          [HALF-OPEN]    │
│       └────────────────────────────────────────┘         │
└──────────────────────────────────────────────────────────┘
```

#### Closed (Normal Operation)

Requests flow through to the downstream service. The circuit breaker tracks the results — counting successes, failures, and timeouts — in a rolling window.

If the **failure rate** (or failure count) exceeds a configured threshold, the circuit trips to **Open**.

```
Rolling window: last 60 seconds
Failure threshold: >50% of requests fail
Minimum request volume: at least 10 requests in window

If failures > 50% AND requests ≥ 10 → trip to OPEN
```

#### Open (Short-Circuit)

All requests **fail immediately** without attempting the downstream call. The caller receives an error (or a fallback response) instantly. No threads are blocked waiting for a slow or dead service.

A timer starts. After a configured **probe interval** (e.g., 10 seconds), the circuit transitions to **Half-Open** to test if the downstream service has recovered.

#### Half-Open (Recovery Probe)

A limited number of **probe requests** are allowed through to the downstream service:
- If the probes **succeed**: the circuit closes — normal operation resumes.
- If the probes **fail**: the circuit trips back to Open; the probe interval resets.

```
Half-Open: allow 3 probe requests
  → 3 successes → [CLOSED]
  → any failure  → [OPEN] (reset probe timer)
```

### Implementation Detail: What Counts as a Failure

A circuit breaker can track:
- **HTTP 5xx responses** (server errors)
- **Timeouts** (no response within the deadline)
- **Network errors** (connection refused, connection reset)
- **Slowness without failure**: count slow calls (latency > threshold) as partial failures, even if they eventually succeed — a slow dependency can be as harmful as a dead one.

Typically, **4xx errors are not counted** as circuit-breaking failures — a 404 or 400 is a client error, not an indication the service is unhealthy.

### Fallback Behavior

When a circuit is open, the caller can:
1. **Return a default/cached value**: serve stale data from a cache.
2. **Return a degraded response**: `{"recommendations": []}` instead of personalized recommendations.
3. **Return an error to the client**: with a descriptive message that retrying won't immediately help.
4. **Queue the request**: attempt it when the circuit closes.

### Circuit Breaker vs Retry

| | Circuit Breaker | Retry with Backoff |
|---|---|---|
| **Purpose** | Prevent further calls to a known-failing service | Retry transient failures |
| **Applies when** | Many calls are failing (systemic failure) | Occasional calls fail (transient failure) |
| **Behavior** | Fail fast, stop calling | Call again, wait between attempts |

They are complementary: use **retries for transient errors** and **circuit breakers for systemic failures**. Without a circuit breaker, retry policies can cascade into DDoS-ing an already-struggling dependency.

### Bulkhead Integration

Circuit breakers are often paired with **bulkheads** (isolated thread pools per dependency). Together:
- The bulkhead limits how many threads can be consumed by one dependency.
- The circuit breaker stops new calls entirely once failure is detected.

---

## Key Trade-offs

| Trade-off | Description |
|---|---|
| **Faster failure vs false trips** | A sensitive threshold (low failure % to trip) catches problems fast but may trip on brief transient hiccups, causing unnecessary outages |
| **Probe frequency** | A short probe interval recovers faster but risks re-tripping if the downstream hasn't fully recovered |
| **Fail-fast vs transparent degradation** | Failing fast protects your system; users experience an error. Returning cached/default values is better UX but may serve stale data |

---

## When to use

Circuit breakers should be placed on **every call to an external dependency**:
- Third-party APIs
- Other internal microservices
- Databases (if connection timeouts are common)
- Caches

They are especially important for synchronous call chains where one failing service can cascade up to user-facing APIs.

---

## Common Pitfalls

- **Not tuning thresholds**: Default thresholds from libraries may be too aggressive (trips on brief spikes) or too lenient (too slow to protect). Monitor and tune based on your baseline error rate.
- **Missing the half-open state**: Implementations that go from Open directly back to Closed without a probe period risk immediately re-tripping as the downstream is still recovering.
- **Counting 4xx as failures**: Client errors (bad input) should not contribute to circuit trip decisions — they're the caller's fault, not the service's.
- **No observable state**: Without metrics and dashboards, you won't know when a circuit is open, which obscures the root cause of downstream problems. Export circuit state as a metric.
- **Circuit breaker + retry without coordination**: Aggressive retries (3 immediate retries) combined with a circuit breaker means 3× the load hits the failing service before the circuit trips. Either reduce retry count, add backoff, or configure the circuit breaker to count retry failures.

# Bulkhead Pattern

## What it is

The **Bulkhead pattern** is a fault-isolation technique that partitions a system's resources into isolated pools, so that a failure or overload in one pool does not cascade and exhaust resources in another. 

The name comes from the compartmentalized watertight compartments in a ship's hull — if one compartment floods, the others remain intact and the ship stays afloat.

In software, bulkheads are commonly implemented as **separate thread pools**, **connection pools**, **process pools**, or **service instance groups** — one per downstream dependency or risk domain.

---

## How it works

### The Problem: Shared Resource Exhaustion

Without bulkheads, all callers share the same thread pool:

```
Service A has a single thread pool: 100 threads

Normal traffic:
  - 70 threads serving product catalog calls  (fast, 10ms)
  - 20 threads serving recommendation calls   (fast, 20ms)
  - 10 threads idle

Payment service goes down (starts returning after 30s timeout):
  - Each payment call holds a thread for 30s
  - After 3 minutes: ALL 100 threads blocked on payments
  - Product catalog calls queue up → timeout → users see failures
  - Entire service is down because ONE dependency failed
```

### Bulkhead Isolation: Thread Pool per Dependency

With bulkheads, each downstream dependency gets its own resource pool:

```
Service A with bulkheads:
  ┌─────────────────────────────────────┐
  │  Product Catalog Thread Pool: 50     │ ← isolated
  │  Recommendation Thread Pool: 30      │ ← isolated
  │  Payment Thread Pool: 20             │ ← isolated
  └─────────────────────────────────────┘

Payment service goes down:
  - Payment pool: 20 threads blocked on payment calls (isolated impact)
  - Product catalog pool: still 50 threads free, serving users normally
  - Recommendation pool: still 30 threads free, operating normally
  - Only payment features are degraded; rest of the service is healthy
```

### Implementation: Bulkhead with Resilience4j (Java)

```java
// Thread pool bulkhead per dependency
BulkheadConfig bulkheadConfig = BulkheadConfig.custom()
    .maxConcurrentCalls(20)           // max concurrent calls to payment service
    .maxWaitDuration(Duration.ofMillis(100))  // if pool full, wait 100ms then fail-fast
    .build();

Bulkhead paymentBulkhead = Bulkhead.of("payment-service", bulkheadConfig);
Bulkhead catalogBulkhead = Bulkhead.of("catalog-service",
    BulkheadConfig.custom().maxConcurrentCalls(50).build());

// Wrap call with bulkhead
Supplier<Order> paymentCall = Bulkhead.decorateSupplier(paymentBulkhead,
    () -> paymentService.charge(order));

Try<Order> result = Try.ofSupplier(paymentCall)
    .recover(BulkheadFullException.class, ex -> {
        log.warn("Payment bulkhead full, returning degraded response");
        return Order.withPendingPayment(order);  // graceful degradation
    });
```

**Thread pool bulkhead (Hystrix/Resilience4j ThreadPoolBulkhead):**
```java
ThreadPoolBulkheadConfig config = ThreadPoolBulkheadConfig.custom()
    .maxThreadPoolSize(10)
    .coreThreadPoolSize(5)
    .queueCapacity(20)      // queue up to 20 requests before rejecting
    .build();

ThreadPoolBulkhead threadPoolBulkhead = ThreadPoolBulkhead.of("payment", config);
```

### Semaphore Bulkhead vs Thread Pool Bulkhead

| | Semaphore Bulkhead | Thread Pool Bulkhead |
|---|---|---|
| **How it works** | Limits concurrent concurrent calls; caller thread is used | Uses separate thread pool; original thread freed |
| **Overhead** | Low | Higher (thread pool management) |
| **Async** | No (caller thread blocked) | Yes (supports futures/async) |
| **Circuit breaker integration** | Natural | Natural |
| **Use case** | In-process calls, low-latency guards | Remote calls, I/O-bound operations |

### Connection Pool Bulkhead

Database connection pools are another common bulkhead boundary:

```yaml
# Separate connection pools per database by domain service
reporting-db:
  maxPoolSize: 20     # large pool for heavy analytics queries

transactional-db:
  maxPoolSize: 50     # separate pool for OLTP — not shared with analytics
  connectionTimeout: 1000ms
```

Without this, a long-running analytics query exhausts all connections, preventing order creation.

### Bulkhead + Circuit Breaker Combination

Bulkheads and circuit breakers are complementary:
- **Bulkhead**: limits concurrency — keeps failures contained.
- **Circuit breaker**: detects failure rate — stops making calls when a service is down.

Combined pattern:
```python
from resilience4j.circuitbreaker import CircuitBreaker
from resilience4j.bulkhead import Bulkhead

# 1. Bulkhead limits concurrent calls to max 20
payment_bulkhead = Bulkhead("payment", max_concurrent_calls=20)

# 2. Circuit breaker opens when 50% of calls fail
payment_circuit_breaker = CircuitBreaker("payment",
    failure_rate_threshold=50,
    wait_duration_in_open_state=30)  # wait 30s before retrying

@payment_bulkhead
@payment_circuit_breaker
def charge_payment(order):
    return payment_service.charge(order)
```

Flow: bulkhead rejects excess concurrent calls; circuit breaker opens when the service is down, failing fast without using any bulkhead capacity.

### Pod/Instance-Level Bulkhead

Bulkheads can also be at the infrastructure level — dedicate specific pods to specific request types:

```
High-priority requests (premium users): dedicated pod group (5 pods)
Standard requests:                       shared pod group (10 pods)
Batch jobs:                              dedicated pod group (3 pods)

If batch jobs consume all CPU:
  → only batch pod group is affected
  → premium and standard pods continue operating normally
```

---

## Key Trade-offs

| Advantage | Disadvantage |
|---|---|
| Failures in one dependency don't cascade | Requires more resources (more threads/connections) |
| Graceful degradation under partial failure | Configuration complexity — pool sizes must be tuned |
| Predictable maximum impact of any single failure | Under-utilization when pools are idle |
| Enables fail-fast instead of slow timeout storms | Adds cognitive overhead to understand resource topology |

---

## When to use

The Bulkhead pattern is essential when:
- A service calls **multiple downstream services** with different reliability profiles — one slow service shouldn't starve calls to a healthy service.
- You need **graceful degradation**: if recommendations are unavailable, product pages should still load.
- You have **different request priorities**: batch/background jobs must not impact interactive user requests.
- You've experienced **cascading failures** where one slow dependency brought down an entire service.

---

## Common Pitfalls

- **Shared thread pool for all external calls**: the most common mistake — all downstream dependencies contend for the same resources. Give each a separate pool.
- **Oversized or undersized pools**: pool too large loses isolation benefits; too small causes excessive rejection under normal load. Baseline using metrics: pool_size ≈ (avg_concurrency × 2) + headroom.
- **Not providing a fallback**: when the bulkhead rejects a call, what does the user see? Always implement a fallback: cached response, default value, graceful error message.
- **Ignoring queue depth**: thread pool bulkheads include a queue. A large queue delays rejection, which can cause latency spikes. Set queue capacity = 0 or very small for latency-sensitive paths.
- **Coupling bulkhead sizing with circuit breaker**: as the circuit breaker opens, fewer calls are made (good). But if the pool size was tuned for open circuit + full load, it may be oversized during steady state. Profile both normal and degraded load conditions.

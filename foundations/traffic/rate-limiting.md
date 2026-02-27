# Rate Limiting

## What it is

**Rate limiting** controls how many requests a client, user, or IP address is allowed to make to a service within a given time window. It is a critical mechanism for:
- **Preventing abuse and DoS attacks** — protecting services from being overwhelmed by malicious or buggy clients.
- **Ensuring fair access** — preventing a single client from monopolizing shared resources.
- **Cost protection** — in metered systems, preventing runaway API usage.
- **Backend protection** — acting as a safety valve to keep downstream services healthy.

The challenge of rate limiting lies in choosing the right algorithm, enforcing limits accurately in a distributed environment, and returning informative responses to throttled clients.

---

## How it works

### Where Rate Limiting Lives

Rate limiting is applied at different layers:

```
Internet → CDN (IP rate limiting) → API Gateway (per-user/per-key) → Service (per-endpoint) → DB (connection limiting)
```

For most purposes, rate limiting at the **API gateway** layer is the right place — it has client identity context (authenticated user ID, API key) and is positioned before any backend work is done.

### Rate Limiting Algorithms

#### 1. Token Bucket

A bucket holds up to **capacity** tokens. Tokens are added at a fixed **refill rate**. Each request consumes one token. If the bucket is empty, the request is rejected.

```
Bucket capacity: 10 tokens
Refill rate: 2 tokens/second

t=0:   Bucket = 10. Client sends 5 requests → Bucket = 5. All allowed.
t=1:   Refill +2. Bucket = 7. Client sends 8 requests → 7 allowed, 1 rejected.
t=2:   Refill +2. Bucket = 2. No requests.
```

**Properties:**
- Allows **burst traffic** up to the bucket capacity.
- Smooth average rate enforcement via the refill rate.
- Widely used because it models real-world usage well — users are allowed occasional bursts.

#### 2. Leaky Bucket

Requests enter a fixed-size queue (the "bucket"). The queue drains (processes requests) at a fixed rate. If the queue is full, incoming requests are dropped.

```
Queue size: 10. Drain rate: 5 req/second.
If 20 requests arrive at once: first 10 enqueue, remaining 10 are dropped.
Requests leave the queue at exactly 5 req/second.
```

**Properties:**
- Enforces a **strictly smooth output rate** — requests flow to the backend at a constant rate.
- Good for protecting backends from bursty traffic.
- Poor for bursty-but-legitimate clients — bursts are absorbed but eventual excess is discarded.

#### 3. Fixed Window Counter

Divide time into fixed windows (e.g., 1-minute intervals). Count requests per window. If count exceeds the limit within the current window, reject.

```
Window: 1 minute. Limit: 100 requests/minute.

00:00–00:59: 100 requests accepted.
01:00–01:59: Counter resets. 100 more accepted.
```

**Problem — Boundary spike**: At 00:59, 100 requests are allowed. At 01:00, the counter resets — another 100 are allowed. In a 2-second window around the boundary, 200 requests pass — 2× the intended rate.

#### 4. Sliding Window Log

Maintain a timestamped log of all requests from a client. On each new request, remove log entries older than the window size, then count remaining entries.

```
Window: 1 minute. Limit: 100 requests/minute.

At t=01:05, remove all log entries before 00:05.
Count remaining entries. If < 100, allow.
```

**Properties:**
- No boundary spike — accurate sliding window.
- High memory cost: stores a timestamp per request per user.

#### 5. Sliding Window Counter

A hybrid of fixed window and sliding window log. Uses two fixed windows (current and previous) and a weighted estimate:

$$\text{Estimated count} = \text{previous\_count} \times \frac{\text{time remaining in previous window}}{\text{window size}} + \text{current\_count}$$

- **Properties**: Low memory cost (two counters per window per user), approximate but accurate enough for most cases.
- Used in Redis-based rate limiters (this formula is used in Redis's rate limiting modules).

### Algorithm Comparison

| Algorithm | Memory | Burst handling | Accuracy | Complexity |
|---|---|---|---|---|
| Token Bucket | Low | Yes | High | Low |
| Leaky Bucket | Low | No (smoothed) | High | Low |
| Fixed Window | Very low | Yes (with boundary spike) | Medium | Very low |
| Sliding Window Log | High | Yes | Exact | Medium |
| Sliding Window Counter | Low | Yes | High (approximate) | Medium |

### Distributed Rate Limiting

In a multi-instance deployment, each instance has its own local counter. Without coordination, a client sending 10 requests split across 10 instances would never be throttled.

**Solutions:**

1. **Centralized store (Redis)**: All instances use atomic Redis operations (`INCR`, `EXPIRE`, `Lua scripts`) to maintain shared counters. The most common approach.
   ```
   PIPELINE:
     INCR   rate:user_42:2026-02-27T10:00
     EXPIRE rate:user_42:2026-02-27T10:00 60
   If counter > limit: reject
   ```

2. **Sticky routing**: Route all requests from the same client to the same instance (via IP hash or session affinity). Avoids coordination overhead but loses flexibility.

3. **Approximate counting with local shards**: Accept slight over-counting — each instance maintains its own counter; periodically sync to a central store. Trade-off between accuracy and coordination cost.

### HTTP Response for Rate Limited Requests

Standard response:
```
HTTP/1.1 429 Too Many Requests
Retry-After: 30
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1709030430
```

Always include `Retry-After` so well-behaved clients back off rather than immediately retrying.

---

## Key Trade-offs

| Trade-off | Description |
|---|---|
| **Accuracy vs complexity** | Sliding window log is exact but memory-heavy; sliding window counter is approximate but efficient |
| **Burst allowance** | Token bucket allows legitimate bursts; leaky bucket never does |
| **Distributed coordination cost** | Centralized Redis counter is accurate but adds latency to every request; local counters are fast but imprecise |

---

## When to use

- **API Gateway level**: enforce per-user/per-key limits before any backend work is done.
- **IP level at CDN/edge**: block blatantly abusive IPs before traffic even reaches your infrastructure.
- **Per-endpoint**: different endpoints may have different limits (write endpoints typically more restricted than reads).
- **Service level**: protect internal services from upstream callers sending too much traffic.

---

## Common Pitfalls

- **Fixed window boundary spike**: use sliding window counter or token bucket to avoid 2× burst at window boundaries.
- **Not returning `Retry-After`**: without it, aggressive clients immediately retry and amplify load.
- **Rate limiting authenticated and unauthenticated requests the same way**: unauthenticated (IP-based) limits should be much stricter; authenticated requests can have higher quotas per user.
- **Forgetting that Redis rate limiting adds latency**: every request incurs a Redis round-trip. Use pipelining and local short-circuit (if counter is clearly below limit, skip Redis) to optimize the hot path.
- **No graceful degradation when rate limit store is down**: if Redis is unavailable, should you allow all traffic (fail-open) or deny all (fail-closed)? Decide deliberately — typically fail-open to preserve availability.

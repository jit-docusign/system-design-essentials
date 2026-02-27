# Latency vs Throughput

## What it is

**Latency** is the time it takes to complete a single operation — from the moment a request is initiated to the moment a response is received. It is a measure of *speed*.

**Throughput** is the number of operations a system can complete per unit of time. It is a measure of *capacity*.

They are related but distinct, and optimizing for one can hurt the other. Understanding both — and which one your system actually needs — is a prerequisite for making sound design decisions.

---

## How it works

### Latency

Latency is typically decomposed into:

| Component | Example |
|---|---|
| Propagation delay | Speed-of-light travel time across a network |
| Transmission delay | Time to push packet bits onto the wire (bandwidth) |
| Processing delay | Time to route/process at each hop |
| Queuing delay | Time waiting in buffers/queues at congested nodes |

**Useful latency reference numbers (approximate):**

| Operation | Latency |
|---|---|
| L1 cache reference | ~1 ns |
| L2 cache reference | ~4 ns |
| Main memory (RAM) access | ~100 ns |
| SSD random read | ~100 µs |
| HDD seek | ~10 ms |
| Network round trip (same datacenter) | ~0.5 ms |
| Network round trip (cross-continent) | ~150 ms |

These numbers matter for back-of-the-envelope reasoning and for understanding where time is being lost.

**Latency percentiles:**

Rather than reporting average latency, production systems report percentiles:
- **P50** — median; half of requests complete faster than this
- **P95** / **P99** — tail latency; the worst 5% / 1% of requests
- **P999** — used when even rare slowness is unacceptable (financial systems)

Averages hide tail latency. A system with P50 = 5ms and P99 = 2000ms has a serious problem for 1 in 100 users.

### Throughput

Throughput is expressed as:
- **Requests per second (RPS)** — for APIs
- **Transactions per second (TPS)** — for databases
- **Bytes per second (Bps)** — for data pipelines and networks
- **Messages per second** — for queues

**Little's Law** connects latency, throughput, and concurrency:

$$L = \lambda \cdot W$$

Where:
- $L$ = average number of requests in the system (queue + processing)
- $\lambda$ = average throughput (arrivals/second)
- $W$ = average latency per request (seconds)

This means: **if you know any two, you can derive the third**.

Example: if a system processes 1,000 RPS and average latency is 50ms, there are on average 50 concurrent requests in-flight at any instant.

### The Latency–Throughput Trade-off

It is often impossible to minimize both simultaneously:

- **Batching** increases throughput but increases latency (you wait to accumulate a batch).
- **Pipelining** increases throughput but can increase per-request latency under contention.
- **Reducing concurrency** decreases queueing latency but reduces throughput.
- **Adding parallelism** increases throughput but adds coordination overhead.

The relationship is often described as:

$$\text{Throughput} \approx \frac{\text{Concurrency}}{\text{Latency}}$$

#### Queue Saturation and the M/M/1 Model

Queueing theory gives a precise explanation for why P99 latency explodes at high utilisation long before a system appears "full".

In an **M/M/1 queue** (single server, Poisson arrivals, exponential service times), the average wait time in queue is:

$$W_q = \frac{\rho}{\mu(1 - \rho)}$$

Where $\rho = \lambda / \mu$ is **utilisation** (arrival rate over service rate).

| Utilisation ($\rho$) | Queue wait (relative to service time) |
|---|---|
| 50% | 1× service time |
| 80% | 4× service time |
| 90% | 9× service time |
| 95% | 19× service time |
| 99% | 99× service time |

As $\rho \to 1$, wait time $\to \infty$. This is why P99 latency starts degrading sharply around 70–80% utilisation, well before you are technically at capacity. **Designing for 100% utilisation is a design for unbounded latency.** Most production systems target 60–70% peak utilisation to keep tail latency predictable.

This model also explains why **bursting matters**: a system that handles average load fine can exhibit severe P99 blowup during a short traffic spike because even a temporary $\rho > 1$ causes the queue to grow without bound until the burst ends.

---

## Key Trade-offs

| Scenario | Trade-off |
|---|---|
| Batching writes to a DB | Throughput ↑, but individual write latency ↑ |
| Increasing thread pool size | Throughput ↑ (up to a point), then queueing and context switching degrades latency |
| Compression | Throughput ↑ (less data on the wire), CPU latency ↑ |
| Synchronous vs async processing | Sync has lower latency for the caller; async enables higher throughput |

---

## When to apply each lens

**Optimize for latency when:**
- User-facing interactions require near-instant feedback (search, checkout, login)
- SLOs specify P99 latency bounds
- Real-time systems where stale or slow data causes harm (trading, fraud detection)

**Optimize for throughput when:**
- The system processes background jobs, ETL pipelines, or log ingestion
- The user doesn't wait synchronously for each individual item
- The bottleneck is volume, not per-operation speed

---

## Common Pitfalls

- **Reporting averages instead of percentiles**: Average latency can look healthy while P99 is terrible. Always look at latency distributions.
- **Ignoring Little's Law**: Under high load, adding threads or replicas may not reduce latency if the service is already bottlenecked on downstream I/O.
- **Confusing network bandwidth with latency**: A high-bandwidth link can still have high round-trip latency. Bandwidth and latency are independent properties.
- **Optimizing throughput at the cost of unacceptable tail latency**: In user-facing systems, a slow P99 is often more damaging than lower average throughput.
- **Not accounting for serialization overhead**: In high-throughput systems, the CPU cost of JSON serialization/deserialization can dominate; switching to binary formats (Protobuf) is a common fix.
- **Coordinated omission in benchmarks**: Most load-testing tools measure the latency of requests they were able to send — they do not account for the requests that were *waiting to be sent* while the system was overloaded. This produces artificially low latency numbers. A benchmark that shows P99 = 20ms under load may be omitting the 30% of requests that were queued for 5 seconds waiting for the tool's thread pool. Gil Tene's work on HdrHistogram and coordinated omission is the canonical reference. When evaluating any benchmark, ask: does this tool account for the full sojourn time, or only the in-flight service time?
- **Designing to 100% utilisation**: As the M/M/1 model shows, latency degrades nonlinearly as utilisation approaches capacity. Target 60–70% peak utilisation for predictable tail latency.

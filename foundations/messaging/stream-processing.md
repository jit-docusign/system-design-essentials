# Stream Processing

## What it is

**Stream processing** is the continuous, real-time computation over an unbounded sequence of events as they arrive — in contrast to **batch processing**, which processes a finite, bounded dataset collected over a period of time.

Instead of waiting for all data to accumulate (batch), stream processing computes results incrementally — updating outputs as each new event arrives. This enables real-time dashboards, fraud detection, alerting, live recommendations, and continuous ETL pipelines.

---

## How it works

### Stream vs Batch Processing

```
Batch Processing:
  ── collect data ──► wait for N hours ──► process entire dataset ──► result

Stream Processing:
  event arrives ──► process immediately ──► update result ──► next event ──► ...
```

| Dimension | Batch | Stream |
|---|---|---|
| Data | Bounded (finite) | Unbounded (infinite) |
| Latency | High (minutes to hours) | Low (milliseconds to seconds) |
| Throughput | Very high | High (with trade-offs) |
| Computation state | Simple — read full dataset | Complex — maintain rolling state |
| Use cases | ETL, reports, ML training | Alerts, fraud, real-time analytics |
| Tools | Spark, Hadoop, dbt | Flink, Kafka Streams, Spark Streaming, Storm |

### Core Concepts

**Stream**: an infinite sequence of events ordered by time.

**Operator**: a transformation applied to a stream — filter, map, aggregate, join.

**State**: accumulated information across multiple events, needed for aggregations, joins, and sequence detection. State is what makes stream processing hard — it must be maintained in memory, checkpointed to disk, and recovered correctly after failures.

**Windowing**: the mechanism for grouping events into finite, bounded subsets for aggregation. Required for computing metrics over time (e.g., "orders per minute").

### Windowing

**Tumbling windows**: fixed-size, non-overlapping intervals.

```
Time: 0───1───2───3───4───5───6  (minutes)
      [── window 1 ──][── window 2 ──]

Events in window 1: [0,2) → count=5
Events in window 2: [2,4) → count=3
```

**Sliding windows**: fixed size, overlap with a configurable slide step.

```
Window size: 2 min, slide: 1 min

[0──2) = events in minutes 0-2
[1──3) = events in minutes 1-3   (overlaps by 1 min)
[2──4) = events in minutes 2-4
```

**Session windows**: variable-size, gap-based. A window closes when no event arrives within the gap duration.

```
Gap threshold: 30 sec

User events: 10:00, 10:05, 10:08, [gap 45s], 11:00, 11:02
Sessions:    [──── session 1 ────]            [── session 2 ──]
```

```python
# Flink: Tumbling window, count orders per product per minute
env = StreamExecutionEnvironment.get_execution_environment()

orders = env.from_kafka("orders")  # source

result = (
    orders
    .filter(lambda e: e['type'] == 'ORDER_PLACED')
    .key_by(lambda e: e['product_id'])
    .window(TumblingEventTimeWindows.of(Time.minutes(1)))
    .aggregate(CountAggregate())
)

result.add_sink(ElasticsearchSink("order-counts-per-min"))
env.execute()
```

### Event Time vs Processing Time

**Processing time**: when the event is processed by the stream processor. Simple but unreliable — network delays cause events to arrive out-of-order.

**Event time**: when the event actually occurred (embedded in the event payload). More accurate but requires handling **late arrivals**.

```
Event generated at:  10:00:05  (event time)
Event arrives at:    10:01:30  (processing time — 85s late due to network)

Window [10:00, 10:01): should this event be included?
  Using processing time: YES (arrived at 10:01:30, within window)
  Using event time: NO (occurred at 10:00:05, in [10:00,10:01) but delayed)
```

The correct answer depends on semantics — event time gives more accurate results but requires waiting for late arrivals.

### Watermarks

A **watermark** is a signal to the stream processor saying: "all events with event_time < W have arrived." It's the mechanism for handling late data.

```
Watermark at W = 10:01:00 means:
  "Any event timestamped before 10:01:00 has arrived (or is considered lost)"

When watermark advances past a window's end time → the window fires and emits results.

Late arrivals after the watermark:
  Option A: Drop the event
  Option B: Update the window result (allowed lateness: e.g., 30s after watermark)
```

```java
// Flink: bounded-out-of-orderness watermark strategy (events arrive up to 5s late)
WatermarkStrategy.<Order>forBoundedOutOfOrderness(Duration.ofSeconds(5))
    .withTimestampAssigner((event, ts) -> event.getTimestamp())
```

### Stateful Computations

**Aggregation**: count, sum, average, min, max over a window.

**Joins**: combine events from two streams with matching keys within a time window.

```
order stream:  { order_id: 1, user_id: A, total: 50 } at 10:00
payment stream: { order_id: 1, amount: 50, status: "ok" } at 10:02

Join on order_id within 5-minute window:
  → enriched event: { order_id: 1, user_id: A, total: 50, payment_status: "ok" }
```

**Stateful pattern detection** — detecting sequences of events:
```
Fraud rule: card charged >3 times in 60 seconds from different countries
  → stream processor maintains per-card state: list of (timestamp, country)
  → on each charge event: check if pattern matches → emit fraud alert
```

State must be **checkpointed** to survive failures:

```
Flink checkpoint:
  Every 30s, all operators snapshot their state to distributed storage (S3/HDFS).
  On failure, restore from last checkpoint → reprocess events from that point.
  Guarantees exactly-once with proper source and sink integration.
```

### Apache Flink vs Kafka Streams vs Spark Streaming

| Feature | Apache Flink | Kafka Streams | Spark Structured Streaming |
|---|---|---|---|
| Deployment | Separate cluster | Embedded in app | Separate cluster |
| Latency | Milliseconds | Milliseconds | Micro-batch (seconds configurable) |
| Exactly-once | Yes (Flink checkpoints) | Yes (Kafka transactions) | Yes (with proper sinks) |
| Event time / watermarks | First-class | Supported | Supported |
| State management | Rich (RocksDB) | RocksDB | Limited |
| Stream-stream joins | Yes | Yes | Yes |
| Use case fit | Complex event processing, CEP | Kafka-native applications | Batch + streaming unified (Spark ecosystem) |

### Lambda Architecture (Batch + Stream)

An older pattern solving: streaming = fast but approximate; batch = slow but accurate.

```
Input ──► Speed Layer (stream) ──► real-time views (approximate)
      ──► Batch Layer (Hadoop) ──► batch views (accurate)
            └──► Serving Layer merges both
```

**Kappa Architecture** replaced lambda in most modern systems: use a stream processor for both real-time and historical reprocessing (replay the event log from Kafka).

---

## Key Trade-offs

| Advantage | Disadvantage |
|---|---|
| Real-time results — milliseconds to seconds | Higher operational complexity than batch |
| Continuous computation — no job scheduling | Stateful operators require careful checkpoint tuning |
| Natural for fraud, alerting, recommendations | Late data and out-of-order events add complexity |
| Kappa: one system for real-time + historical | Stream processors are harder to debug than batch jobs |
| Scales horizontally with event volume | Windowing semantics require careful design |

---

## When to use

- **Fraud detection and anomaly detection**: detect suspicious patterns in milliseconds.
- **Real-time dashboards**: clicks/second, active users, live revenue.
- **Continuous ETL**: transform, enrich, and route events as they arrive — no nightly batch jobs.
- **IoT sensor monitoring**: trigger alerts when sensor readings exceed thresholds.
- **Personalization and recommendations**: update user profiles and serve fresh recommendations.
- **Financial risk management**: monitor positions and PnL in real-time during market hours.

Batch is still appropriate for:
- Large historical data processing (ad-hoc analytics).
- ML model training.
- Complex SQL queries over billions of rows.

---

## Common Pitfalls

- **Ignoring late data**: not configuring watermarks or allowed lateness causes windows to never close or results to be incorrect. Always choose a watermark strategy that matches your source's late-arrival characteristics.
- **Too much state without backpressure tuning**: unbounded state accumulates until the operator runs out of memory. Monitor state size and configure TTL on state entries.
- **Not checkpointing**: without checkpointing, a node failure restarts processing from the latest offset with no state recovery, causing incorrect aggregations. Configure checkpoints.
- **Processing time semantics for accuracy**: using processing time for SLA calculations or billing is inaccurate because events arrive late. Use event time.
- **Mixing low-latency SLAs with complex joins**: multi-stream joins with wide windows are expensive. Profile and tune parallelism carefully to meet latency targets.
- **Under-partitioning the source**: Kafka partitions limit parallelism in Kafka Streams and Flink Kafka sources. Ensure source topics are partitioned adequately before deploying at scale.

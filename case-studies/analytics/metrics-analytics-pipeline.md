# Design a Metrics and Analytics Pipeline

## Problem Statement

Design a data pipeline that collects user events (page views, clicks, purchases, errors) from millions of client applications, processes them in near-real-time, and makes them available for dashboards and analysis. This models how systems like Google Analytics, Mixpanel, Amplitude, and internal analytics at Netflix/Uber work.

**Scale requirements**:
- 1 million events per second at peak
- 100 TB of events per day
- Real-time dashboard: data visible within 30 seconds of event
- Historical queries: scan years of data in under 30 seconds
- Queryable retention: 90 days raw; 2 years aggregated

---

## Architecture (Lambda vs. Kappa)

### Lambda Architecture (Batch + Speed Layer)

```
Events
  │
  ├──────────── Speed Layer (real-time) ────────────┐
  │             Apache Flink / Kafka Streams         │
  │             30-second windowed aggregations      │
  │             → Real-time OLAP (Druid/ClickHouse) │
  │                                                  │
  └──────────── Batch Layer ─────────────────────────┤
               Apache Spark (hourly/daily jobs)      │
               S3 Parquet (raw event store)          │
               → Historical OLAP (BigQuery/Redshift)  │
                                                     │
                             Serving Layer ◄─────────┘
                             Query API → Dashboard
```

**Lambda pros**: batch layer is source of truth (can reprocess if streaming has bugs)  
**Lambda cons**: two separate codebases (stream + batch), complex to maintain

### Kappa Architecture (Stream-Only)

```
Events
  │
  ▼
Kafka (durable log, 90 days retention)
  │
  ├── Flink Job (real-time aggregations) → ClickHouse (OLAP)
  │
  └── Flink Job (bulk reprocessing, same code) → ClickHouse (historical)

One codebase for both real-time and historical.
Reprocess by replaying Kafka from beginning.
```

**Kappa pros**: simpler (one system, one codebase)  
**Kappa cons**: Kafka retention is expensive; streaming has unique challenges vs batch

---

## How It Works

### 1. Event Collection (Ingest Layer)

```python
# Client SDK sends events to Collection Service
# Batched client-side (to reduce HTTP overhead)

class EventBatch:
    """Client SDK batches events before sending"""
    
    def __init__(self, flush_interval_ms=1000, max_batch_size=100):
        self.queue = []
        self.flush_interval = flush_interval_ms
        self.max_size = max_batch_size
    
    def track(self, event_type: str, properties: dict, user_id: str):
        event = {
            "event_id": str(uuid.uuid4()),
            "event_type": event_type,
            "user_id": user_id,
            "session_id": self.get_session_id(),
            "timestamp": int(time.time() * 1000),  # ms UTC
            "client_time": datetime.utcnow().isoformat(),
            "properties": properties,
            "context": {
                "app_version": APP_VERSION,
                "os": platform.system(),
                "screen_resolution": f"{screen_w}x{screen_h}",
                "locale": locale.getdefaultlocale()[0]
            }
        }
        self.queue.append(event)
        
        if len(self.queue) >= self.max_size:
            self.flush()
    
    def flush(self):
        if not self.queue:
            return
        
        batch = self.queue[:]
        self.queue.clear()
        
        # POST to collection endpoint
        try:
            requests.post(
                "https://collect.analytics.example.com/events",
                json={"events": batch},
                timeout=5,
                headers={"X-API-Key": API_KEY}
            )
        except Exception:
            # Re-queue on failure (with retry limit to avoid memory leak)
            self.queue[:0] = batch[:50]  # re-queue up to 50 events

# Usage:
analytics = EventBatch()
analytics.track("page_view", {"page": "/checkout", "product_id": "abc"}, user_id="u42")
analytics.track("button_click", {"button_id": "add-to-cart"}, user_id="u42")
```

### 2. Collection Service and Kafka

```python
# Collection Service: HTTP → Kafka producer

from kafka import KafkaProducer
import json

producer = KafkaProducer(
    bootstrap_servers=["kafka1:9092", "kafka2:9092", "kafka3:9092"],
    value_serializer=lambda v: json.dumps(v).encode("utf-8"),
    compression_type="snappy",     # compress batches
    batch_size=65536,              # 64 KB batches
    linger_ms=10,                  # wait 10ms to accumulate batch
    acks="all",                    # wait for all replicas to acknowledge
    retries=3
)

async def collect_events(request):
    events = await request.json()
    
    # Validate, enrich, and publish each event
    for event in events.get("events", []):
        enriched = {
            **event,
            "server_time": int(time.time() * 1000),
            "ip": hash_ip(request.client.host),  # hash for privacy
            "geo": geoip_lookup(request.client.host),  # city/country
        }
        
        # Route to appropriate Kafka topic
        topic = f"events.{event['event_type']}"  # e.g., "events.page_view"
        
        producer.send(
            topic,
            key=event.get("user_id", "anonymous").encode(),  # partition by user_id
            value=enriched
        )
    
    producer.flush()
    return {"status": "ok", "accepted": len(events.get("events", []))}
```

### 3. Kafka Topic Design

```
Topics:
  events.page_view        → partition by user_id (user's page views in order)
  events.click            → partition by user_id
  events.purchase         → partition by user_id
  events.error            → partition by app_version (cluster errors by release)
  events.raw_all          → all events merged (for batch processing)

Configuration:
  - Partitions: 200 per topic (1M events/sec ÷ 5K events/sec/partition)
  - Replication factor: 3 (one leader + two replicas per partition)
  - Retention: 7 days (Kafka as buffer), long-term in S3
  - Message format: JSON compressed with Snappy (or Avro with schema registry)

Schema evolution (Avro + Confluent Schema Registry):
  - Events have strict schema; new fields are backwards-compatible
  - Schema registry prevents breaking changes to event format
```

### 4. Stream Processing with Apache Flink

```python
# Flink job: compute real-time metrics
# (Pseudocode — actual Flink uses Java/Scala/Python API)

from pyflink.datastream import StreamExecutionEnvironment
from pyflink.datastream.connectors.kafka import KafkaSource, KafkaSink

env = StreamExecutionEnvironment.get_execution_environment()

# Source: consume from Kafka
source = (KafkaSource.builder()
    .set_bootstrap_servers("kafka1:9092")
    .set_topics("events.page_view")
    .set_value_only_deserializer(JsonDeserializationSchema())
    .build())

stream = env.from_source(source, watermark_strategy, "Kafka Source")

# Transformation 1: Tumbling window — events per page per minute
page_views_per_minute = (
    stream
    .map(lambda e: (e["properties"]["page"], 1))  # (page, 1)
    .key_by(lambda x: x[0])                        # group by page
    .window(TumblingEventTimeWindows.of(Time.minutes(1)))
    .reduce(lambda a, b: (a[0], a[1] + b[1]))      # sum counts
)

# Transformation 2: Session analytics
# Define user session as events within 30 min of each other
user_sessions = (
    stream
    .key_by(lambda e: e["user_id"])
    .window(SessionWindows.with_gap(Time.minutes(30)))
    .process(SessionAggregateFunction())  # first_event, last_event, event_count
)

# Sink: write to ClickHouse for real-time OLAP queries
sink = ClickHouseSink.builder()...build()
page_views_per_minute.sink_to(sink)

# Also sink raw events to S3 for batch processing
s3_sink = FileSink.for_bulk_format(
    "s3://analytics-events/",
    ParquetWriterFactory(EventSchema)
).with_rolling_policy(
    OnCheckpointRollingPolicy.build()  # new file per checkpoint (every 60s)
).build()

stream.sink_to(s3_sink)
```

### 5. OLAP Storage — ClickHouse

ClickHouse is a column-oriented database optimized for analytical queries on time-series data:

```sql
-- ClickHouse table for page_view events
CREATE TABLE page_views (
    event_id        UUID,
    user_id         String,
    session_id      String,
    page            String,
    timestamp       DateTime64(3),   -- millisecond precision
    country         LowCardinality(String),
    app_version     LowCardinality(String),
    device_type     LowCardinality(String)
) ENGINE = MergeTree()
  PARTITION BY toYYYYMM(timestamp)    -- partition by month for pruning
  ORDER BY (page, timestamp)          -- primary sort key: queries filter by page
  TTL timestamp + INTERVAL 90 DAY;   -- auto-delete after 90 days

-- Materialized view: pre-aggregate counts per minute (for fast dashboard)
CREATE MATERIALIZED VIEW page_views_by_minute
ENGINE = SummingMergeTree()
PARTITION BY toYYYYMM(minute)
ORDER BY (page, minute)
AS SELECT
    page,
    toStartOfMinute(timestamp) AS minute,
    count() AS view_count,
    uniq(user_id) AS unique_users
FROM page_views
GROUP BY page, minute;

-- Dashboard query: views per page in last 24 hours (hits materialized view)
SELECT page, sum(view_count) AS total_views
FROM page_views_by_minute
WHERE minute >= now() - INTERVAL 24 HOUR
GROUP BY page
ORDER BY total_views DESC
LIMIT 10;
-- Executes in ~100ms even over billions of events
```

---

## Real-Time vs. Batch Trade-offs

| | Real-time (Flink + ClickHouse) | Batch (Spark + BigQuery) |
|---|---|---|
| Latency | 30 seconds | Hours |
| Accuracy | Approximate (late events) | Exact |
| Cost | High (always-on cluster) | Low (pay-per-query) |
| Complexity | High | Lower |
| Use for | Live dashboards, alerts | Historical reports, ML training |

### Late Event Handling

```
Event created at 14:00:00 on device, but arrives at server at 14:02:30 (offline for 2 min)
Flink's 14:00 - 14:01 window has already closed!

Solutions:
  1. Watermarks: Flink processes events up to (current_time - 2 min) late
     Allow events up to 2 minutes late; discard older late arrivals
  
  2. Lambda fallback: batch layer reprocesses with exact event_time (not server_time)
  
  3. Accept approximate real-time; exact numbers come from batch
     Dashboard shows: "~1,234 views (updated 5 seconds ago)"
     Daily report shows: "1,247 views (exact, includes late events)"
```

---

## Hard-Learned Engineering Lessons

1. **Not distinguishing event time from processing time**: events arrive late (mobile offline, network lag). Using server ingestion time for analytics gives wrong results. Always record `client_timestamp` and process with it.

2. **Forgetting deduplication at ingest**: clients retry failed publishes. Duplicate events (same `event_id`) must be deduplicated before writing to OLAP. Use `event_id` as idempotency key.

3. **Not addressing fan-out hotspots**: Kafka partition by `user_id` is fine but can create hot partitions for power users who generate 100× average events. Consider partition by `event_id` hash for uniform distribution, with separate stream for per-user analytics.

4. **Treating real-time and batch as alternatives**: they're complementary. Real-time for dashboards; batch for exact historical reports. Lambda/Kappa are both valid; know the trade-offs.

5. **Not addressing schema evolution**: event schemas change. Without Avro + Schema Registry, adding a new field to events can break consumers. Discuss backwards-compatible schema changes.

6. **Forgetting data privacy**: raw events contain PII (user IDs, IPs, locations). Discuss: hashing IPs at ingestion, user deletion (GDPR right-to-erasure), and data residency (EU events stay in EU).

7. **Missing the serving layer**: raw ClickHouse queries are fast but dashboard requires pre-aggregated materialized views for sub-second response. Materialized views are the key to scalable analytics dashboards.

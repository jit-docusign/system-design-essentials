# Time-Series Databases

## What it is

A **time-series database (TSDB)** is a database optimized for storing, querying, and analyzing **sequences of data points indexed by time**. Unlike general-purpose databases, TSDBs are purpose-built for the specific patterns of time-ordered, append-heavy workloads — insert-intensive, rarely updated, and almost always queried as ranges over time.

Every measurement in a TSDB is associated with a **timestamp + value + optional tags/labels**.

**Common systems**: InfluxDB, TimescaleDB (PostgreSQL extension), Prometheus (TSDB + monitoring), VictoriaMetrics, Apache Druid, QuestDB, Amazon Timestream.

---

## How it works

### Data Model

Time-series data has a canonical structure:

```
metric_name{label_key="label_value", ...} value  timestamp

# Prometheus exposition format examples:
http_requests_total{method="POST", status="200", endpoint="/api/orders"} 4523  1720000000
http_requests_total{method="GET",  status="200", endpoint="/api/users"}  8901  1720000000
cpu_usage_percent{host="web-01", region="us-east"}                       72.4  1720000000
```

A **time series** is a unique combination of metric name and label set. Two series with different labels are stored independently:
```
cpu_usage_percent{host="web-01"}   -- one time series
cpu_usage_percent{host="web-02"}   -- separate time series
```

**InfluxDB line protocol:**
```
measurement,tag1=val1,tag2=val2 field1=v1,field2=v2 timestamp_nanoseconds

cpu_usage,host=web-01,region=us-east usage_idle=27.6,usage_system=4.1 1720000000000000000
```

### Storage Optimization for Time-Series

General-purpose databases are inefficient for time-series because:
- Data is **always inserted at the current time** (monotonically increasing timestamps) — random insert patterns don't occur.
- **Old data is rarely updated** — only new data is written.
- **Queries are range-based** over time and labels — not single-row lookups.

TSDBs exploit these properties:

**1. Columnar storage with delta-encoding:**
Timestamps and values are stored in separate columns. Timestamps form a nearly monotonic sequence — storing deltas (differences) rather than absolute values achieves 10–50× compression.

**2. Gorilla compression (Facebook's algorithm, used in many TSDBs):**
Double-precision float values are XOR'd with the previous value. Sensor readings that change slightly store only the XOR delta in a few bits rather than a full 64-bit float.

**3. Time-partitioned storage:**
Data is automatically partitioned into time buckets (chunks). Old, cold chunks are compressed aggressively. Hot current chunks remain in memory.

```
TimescaleDB "hypertable" internals:
  chunk: 2024-01-01 to 2024-01-07   (compressed, on disk)
  chunk: 2024-01-08 to 2024-01-14   (compressed, on disk)
  chunk: 2024-01-15 to 2024-01-21   (hot, in memory)
```

**4. Automatic data retention (TTL):**
TSDBs support configurable retention policies — e.g., keep raw data for 30 days, downsampled (1-min averages) for 1 year, 1-hour averages forever.

### Query Patterns

Time-series queries are range + aggregation operations:

```sql
-- TimescaleDB (PostgreSQL syntax):

-- Rolling 5-minute average CPU per host
SELECT
  time_bucket('5 minutes', time) AS bucket,
  host,
  AVG(cpu_usage) AS avg_cpu
FROM metrics
WHERE time > NOW() - INTERVAL '1 hour'
GROUP BY bucket, host
ORDER BY bucket DESC;

-- Downsampling — 1-hour averages
SELECT
  time_bucket('1 hour', time) AS hour,
  AVG(value), MAX(value), MIN(value)
FROM measurements
WHERE time BETWEEN '2024-01-01' AND '2024-02-01'
GROUP BY hour;
```

**PromQL (Prometheus Query Language):**
```promql
# 5-minute rate of HTTP requests per second
rate(http_requests_total{status="200"}[5m])

# 95th percentile response latency
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))

# Alert: CPU over 90% for 5 minutes
avg by (host)(cpu_usage_percent) > 90
```

### Downsampling and Rollup

Storing every raw data point forever is expensive. Downsampling aggregates high-frequency raw data into lower-resolution summaries:

```
Raw data: one point per second  → keep for 7 days
1-minute aggregates            → keep for 90 days
1-hour aggregates              → keep for 1 year
1-day aggregates               → keep forever
```

This is called a **retention and downsampling policy** — automatically maintained by the TSDB.

### Prometheus Architecture

Prometheus is the standard monitoring TSDB for cloud-native environments:

```
Targets (apps, infra)
  │ /metrics endpoint (HTTP pull)
  │
  ▼
Prometheus Server
  ├─ Scraper: polls /metrics every 15s
  ├─ TSDB: local storage of scraped data
  ├─ Rule Engine: alerting rules → Alertmanager
  └─ PromQL API → Grafana (dashboards)
```

Prometheus uses a **pull model** — it scrapes targets rather than receiving pushed metrics. This makes failed scrapes detectable (useful for alerting on missing data).

---

## Key Trade-offs

| Advantage | Disadvantage |
|---|---|
| Extremely high ingest rate (millions points/sec) | Poor for non-time-ordered queries |
| High compression for time-ordered data | Not a general-purpose database |
| Range queries + aggregations are native | Limited join capabilities |
| Automatic TTL and downsampling | Cardinality explosion with too many label dimensions |
| Purpose-built query languages (PromQL) | Ecosystem lock-in for proprietary formats |

---

## When to use

Time-series databases are the right choice for:
- **Infrastructure monitoring**: CPU, memory, network, disk metrics.
- **Application performance monitoring (APM)**: request rate, latency, error rate.
- **IoT sensor data**: temperature, pressure, GPS coordinates, power consumption.
- **Financial data**: stock prices, transaction volumes, exchange rates.
- **Business metrics**: DAU, revenue per hour, conversion rates over time.

Rule of thumb: if the primary access pattern is "give me all values of X between time A and time B", a TSDB is likely the right choice.

---

## Common Pitfalls

- **High cardinality label explosions**: if a label value is highly unique (e.g., `request_id`, `user_id`, `trace_id`), each unique combination creates a separate time series. Millions of unique time series (high cardinality) degrade performance and memory usage drastically. Keep label values bounded.
- **Storing events as time-series**: time-series stores are for numeric measurements. For structured events or logs, use a log management system (Elasticsearch, Loki) instead.
- **Not configuring retention policies**: without retention policies, TSDBs grow indefinitely and exhaust storage. Always configure raw data TTL and downsampling.
- **Using general-purpose database for time-series**: storing millions of timestamped rows in a standard relational database without TSDB-specific optimizations leads to poor compression and slow range aggregations. If the workload is time-series, use a purpose-built TSDB.

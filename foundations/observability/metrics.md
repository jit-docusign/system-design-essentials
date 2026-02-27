# Metrics

## What it is

**Metrics** are numeric measurements collected over time that describe the behavior of a system — request rates, error rates, latency percentiles, CPU usage, queue depth. Unlike logs (discrete events), metrics are **aggregated continuously** and are the primary tool for monitoring system health, capacity planning, and alerting.

---

## How it works

### Metric Types

| Type | Description | Use Case | Example |
|---|---|---|---|
| **Counter** | Monotonically increasing value; only goes up (resets on restart) | Total requests, errors, bytes sent | `http_requests_total` |
| **Gauge** | Current value, can go up or down | CPU usage, active connections, queue depth | `active_connections` |
| **Histogram** | Distribution of observed values across configurable buckets | Request latency, payload size | `http_request_duration_seconds` |
| **Summary** | Pre-computed quantiles (streaming) | Client-side percentile tracking | `rpc_duration_p99` |

### Prometheus (Pull-Based Collection)

Prometheus scrapes metrics from instrumented services on a schedule:

```
┌─────────────┐  /metrics   ┌───────────────┐
│ Service A   │◄───────────│               │
│  :8080      │            │  Prometheus   │──── Query ────► Grafana
├─────────────┤  /metrics   │  Server       │
│ Service B   │◄───────────│               │
│  :8080      │            └───────────────┘
└─────────────┘
     ▲
     │ Python instrumentation
```

```python
from prometheus_client import Counter, Histogram, Gauge, start_http_server
import time

# Define metrics with labels
REQUEST_COUNT = Counter(
    'http_requests_total',
    'Total HTTP requests',
    ['method', 'endpoint', 'status_code']
)

REQUEST_LATENCY = Histogram(
    'http_request_duration_seconds',
    'HTTP request latency',
    ['method', 'endpoint'],
    buckets=[0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0]
)

ACTIVE_REQUESTS = Gauge(
    'http_requests_active',
    'Number of requests currently being processed'
)

DB_CONNECTIONS = Gauge(
    'db_connection_pool_size',
    'Database connection pool metrics',
    ['state']  # label: 'active' | 'idle' | 'waiting'
)

# Use in your handler
def handle_request(method: str, endpoint: str):
    ACTIVE_REQUESTS.inc()
    start = time.time()
    status = 'unknown'
    try:
        result = process()
        status = '200'
        return result
    except Exception:
        status = '500'
        raise
    finally:
        duration = time.time() - start
        REQUEST_COUNT.labels(method=method, endpoint=endpoint, status_code=status).inc()
        REQUEST_LATENCY.labels(method=method, endpoint=endpoint).observe(duration)
        ACTIVE_REQUESTS.dec()

# Expose /metrics endpoint
start_http_server(8080)
```

```
# Example /metrics output (Prometheus text format):
http_requests_total{method="GET",endpoint="/api/orders",status_code="200"} 12453
http_requests_total{method="GET",endpoint="/api/orders",status_code="500"} 17
http_request_duration_seconds_bucket{le="0.05",endpoint="/api/orders"} 11200
http_request_duration_seconds_bucket{le="0.1",endpoint="/api/orders"} 12100
http_request_duration_seconds_bucket{le="0.25",endpoint="/api/orders"} 12410
http_request_duration_seconds_sum{endpoint="/api/orders"} 845.32
http_request_duration_seconds_count{endpoint="/api/orders"} 12453
```

### PromQL (Prometheus Query Language)

```promql
# Request rate per second (5-min window)
rate(http_requests_total[5m])

# Error rate as a fraction of total requests
rate(http_requests_total{status_code=~"5.."}[5m])
  /
rate(http_requests_total[5m])

# 99th percentile latency
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))

# Alert: error rate > 1% for 5 minutes
- alert: HighErrorRate
  expr: |
    rate(http_requests_total{status_code=~"5.."}[5m])
    / rate(http_requests_total[5m]) > 0.01
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "Error rate above 1% on {{ $labels.endpoint }}"
    runbook: "https://runbooks.example.com/high-error-rate"
```

### The Four Golden Signals (SRE Playbook)

Google SRE defines four signals that, if monitored, give broad coverage of service health:

| Signal | Definition | Metric Example |
|---|---|---|
| **Latency** | Time to serve a request (separate success vs error latency) | p50, p95, p99 of `request_duration_seconds` |
| **Traffic** | Demand on the system | `rate(http_requests_total[1m])` |
| **Errors** | Rate of failed requests (explicit 5xx, implicit wrong results) | `rate(errors_total[5m]) / rate(requests_total[5m])` |
| **Saturation** | How "full" the service is (CPU, memory, queue depth) | `container_cpu_usage_seconds_total`, queue lag |

### USE Method (for Resources)

For every physical/virtual resource:

| Signal | Description |
|---|---|
| **Utilization** | % time resource is busy (CPU, disk I/O) |
| **Saturation** | Amount of queued/waiting work |
| **Errors** | Error rate for the device/resource |

```promql
# CPU utilization
1 - avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) by (node)

# Network saturation (packets dropped)
rate(node_network_receive_drop_total[5m])

# Disk utilization
rate(node_disk_io_time_seconds_total[5m])
```

### RED Method (for Services)

For every microservice:

| Signal | Definition |
|---|---|
| **Rate** | Requests per second |
| **Errors** | Error rate (absolute or fraction) |
| **Duration** | Latency distribution (p50/p95/p99) |

Start with RED for application-level health, fall back to USE for resource-level investigation.

### Histograms vs Summaries

| Dimension | Histogram | Summary |
|---|---|---|
| Quantiles computed | Server-side (PromQL `histogram_quantile`) | Client-side |
| Aggregatable | Yes — aggregate across replicas | No — cannot average percentiles across instances |
| Configuration | Define buckets at instrument time | Define quantiles at instrument time |
| Use when | Multiple replicas, need aggregated percentiles | Single instance or pre-defined quantiles |

**Recommendation**: Almost always use Histograms with Prometheus. Summaries cannot be meaningfully aggregated across pod replicas.

### Cardinality Management

Metric cardinality = number of unique label value combinations. High cardinality kills Prometheus:

```python
# BAD: user_id as a label — creates millions of time series
REQUEST_COUNT = Counter('requests_total', '...', ['endpoint', 'user_id'])  # DON'T DO THIS

# GOOD: bounded, low-cardinality labels
REQUEST_COUNT = Counter('requests_total', '...', ['endpoint', 'method', 'status_code'])
# endpoint: ~50 values, method: ~5 values, status: ~10 values → 2500 time series max
```

Rule of thumb: keep per-metric cardinality under 10,000 time series. Use logs for high-cardinality data (user IDs, request IDs, order IDs).

### Push vs Pull Collection

| Model | Description | Tools |
|---|---|---|
| **Pull** | Metrics server scrapes services | Prometheus |
| **Push** | Services push to a collector | StatsD, InfluxDB, Datadog Agent, OpenTelemetry |

Prometheus pull model advantages:
- Prometheus controls scrape interval — consistent data.
- Easier to detect dead services (scrape fails).
- No firewall holes needed (Prometheus reaches into the cluster).

Push model advantages:
- Works behind NAT / firewalls.
- Better for short-lived batch jobs (use `pushgateway` in Prometheus for this).

---

## Key Trade-offs

| Dimension | Metrics | Logs |
|---|---|---|
| Volume | Low (aggregated numbers) | High (one entry per event) |
| Query speed | Very fast | Slower (full-text search) |
| Granularity | Aggregated (lose individual events) | Individual event detail |
| Alerting | Ideal (numeric thresholds) | Possible but heavyweight |
| Cost at scale | Low | High |

---

## When to use

- Use metrics for **alerting** (numeric thresholds are fast to evaluate).
- Use metrics for **dashboards** and capacity planning (long-term time series).
- Combine with logs for drilling into individual errors after detecting them in metrics.
- Combine with distributed tracing for finding where in a request's lifecycle latency occurs.

---

## Common Pitfalls

- **Alerting on averages**: average latency hides tail latency. Alert on p95 or p99. A service with p50=10ms and p99=10s looks fine on average but 1% of users wait 10 seconds.
- **High-cardinality labels**: adding `user_id` or `request_id` as metric labels creates one time series per user/request. Prometheus can't handle millions of time series. Use histogram buckets for distributions, logs for individual values.
- **Measuring only the happy path**: instrument error paths explicitly. Don't rely on absence of success metrics to detect failures — add error counters.
- **Not bucketing histograms appropriately**: default Prometheus histogram buckets (0.005s to 10s) may not match your SLOs. Choose buckets around your SLO thresholds (e.g., if your SLO is 200ms, put a bucket at exactly 0.2).
- **Scrape interval too long**: Prometheus default is 15s. For latency-sensitive alerting, use 10s. For very high-frequency events, consider increasing resolution rather than just raw scrape interval.

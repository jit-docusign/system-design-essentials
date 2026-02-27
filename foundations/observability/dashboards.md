# Dashboards and Observability Dashboards

## What it is

**Dashboards** are visual representations of metrics, logs, and traces that give teams a shared, at-a-glance understanding of system health. They serve two purposes:
- **Operational**: real-time monitoring for on-call engineers responding to incidents.
- **Analytical**: trend analysis, capacity planning, SLO tracking over longer time windows.

**Grafana** is the dominant open-source dashboarding platform, supporting dozens of data sources (Prometheus, Elasticsearch, Loki, Tempo, PostgreSQL, CloudWatch, and more).

---

## How it works

### Dashboard Hierarchy

Organize dashboards in layers — each layer serves a different audience and answers different questions:

```
├── 1. Service Health Overview  (for on-call, leadership)
│       "Is everything OK right now?"
│       Metrics: SLO status, error rates, request rates
│
├── 2. Service Detail Dashboard  (for on-call engineers)
│       "Why is the order service degraded?"
│       Metrics: RED (Rate/Error/Duration) per endpoint
│
└── 3. Infrastructure Dashboard  (for platform/SRE)
        "What is the underlying cause? CPU/Memory/Network?"
        Metrics: USE (Utilization/Saturation/Errors) per resource
```

### Grafana Dashboard Structure

```json
// Core concepts in a Grafana dashboard JSON
{
  "title": "Order Service",
  "uid": "order-service-v2",
  "time": {"from": "now-1h", "to": "now"},
  "refresh": "30s",
  "panels": [
    // Row: Overview (request rate, error rate, latency)
    // Row: Per-Endpoint Breakdown
    // Row: Database Metrics
    // Row: Infrastructure
  ],
  "templating": {
    "list": [
      {"name": "env", "query": "label_values(environment)"},          // dropdown: prod/staging
      {"name": "service_instance", "query": "label_values(instance)"} // dropdown: pod instances
    ]
  }
}
```

### Key Panel Types

| Panel Type | Data Source | Use For |
|---|---|---|
| **Time series** | Prometheus | Rate, latency trends over time |
| **Stat** | Prometheus | Current value at a glance (SLO status, error count) |
| **Table** | Any | Per-endpoint breakdown, sorted by error rate |
| **Gauge** | Prometheus | Percentages (disk usage, error budget consumed) |
| **Heatmap** | Prometheus histogram | Latency distribution over time |
| **Logs panel** | Loki | Recent log lines, filtered by trace/request |
| **Node graph** | Tempo/Jaeger | Service dependency map with error rates |

### Service Health Dashboard (SLO Dashboard)

The most important dashboard — answers "are we meeting our commitments?"

```
┌────────────────────────────────────────────────────────────┐
│  Order Service — SLO Dashboard                     🟢 OK   │
├──────────────┬─────────────────┬──────────────────────────┤
│              │  Error Budget    │ 30-Day Window             │
│  Availability│  ██████████░░   │ 78% consumed (3.2 days    │
│  99.97%      │  22% remaining  │ remaining)                │
├──────────────┴─────────────────┴──────────────────────────┤
│             Request Rate [/s]           Error Rate [%]     │
│  450│        ___                0.5│                       │
│  300│       /   \               0.3│                       │
│  150│  _   /     \_    __       0.1│____________________   │
│    0│ / \/          \__/          0│                       │
│       1h ago        now           0  1h ago      now       │
├────────────────────────────────────────────────────────────┤
│           Latency Percentiles              P99 = 157ms      │
│  600│                                                       │
│  400│                  P99 ‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾   │
│  200│  P95 _____________                                    │
│    0│  P50                                                  │
└────────────────────────────────────────────────────────────┘
```

```yaml
# Grafana-as-code (Grafonnet / JSONNET or provisioning YAML)
# panels:
- title: Availability (30-day SLO)
  type: stat
  targets:
    - expr: |
        1 - (
          sum_over_time(rate(http_requests_total{status=~"5.."}[5m])[30d:5m])
          / sum_over_time(rate(http_requests_total[5m])[30d:5m])
        )
  options:
    reduceOptions: {calcs: ['lastNotNull']}
    thresholds:
      steps:
        - color: red; value: 0          # < SLO
        - color: yellow; value: 0.999   # at SLO boundary
        - color: green; value: 0.9995   # healthy margin

- title: Error Rate %
  type: timeseries
  targets:
    - expr: |
        100 * rate(http_requests_total{status=~"5.."}[5m])
        / rate(http_requests_total[5m])
      legendFormat: "{{endpoint}}"
  fieldConfig:
    defaults:
      thresholds:
        steps: [{color: green, value: 0}, {color: red, value: 1}]
```

### Latency Heatmap (Histogram Visualization)

A heatmap shows the full distribution of latency over time — better than a single percentile line:

```
Latency Heatmap: http_request_duration_seconds
─────────────────────────────────────────────────
Time →
500ms│                        ████
250ms│               ██████████████████
100ms│    ██████████████████████████████████
 50ms│  ██████████████████████████████████████
 10ms│████████████████████████████████████
      12:00    12:15    12:30    12:45

Darker = more requests at that latency bucket.
Sudden vertical band of dark at 500ms = latency spike.
```

```yaml
- title: Request Latency Heatmap
  type: heatmap
  targets:
    - expr: sum by (le) (rate(http_request_duration_seconds_bucket[5m]))
      legendFormat: "{{le}}"
      format: heatmap
```

### Exemplars: Linking Metrics to Traces

Prometheus exemplars embed a trace_id alongside a metric data point. In Grafana, clicking an outlier data point opens the corresponding trace:

```python
import prometheus_client as prom
from opentelemetry import trace

REQUEST_LATENCY = prom.Histogram(
    'http_request_duration_seconds', 'Latency',
    buckets=[0.05, 0.1, 0.25, 0.5, 1.0, 2.5]
)

def handle_request():
    start = time.time()
    current_span = trace.get_current_span()
    span_context = current_span.get_span_context()
    
    # Do work...
    
    duration = time.time() - start
    REQUEST_LATENCY.observe(
        duration,
        exemplar={'trace_id': format(span_context.trace_id, '032x')}
    )
    # In Grafana: hover over a spike → click "View Exemplar" → Tempo opens the trace
```

### Runbook Embedding

Every dashboard panel that represents an SLO or alert should link to its runbook:

```json
{
  "title": "High Error Rate",
  "description": "See [Runbook](https://runbooks.example.com/order-service/high-error-rate) | [PagerDuty](https://pagerduty.example.com) if alert is firing",
  "links": [
    {"title": "Runbook", "url": "https://runbooks.example.com/order-service/high-error-rate"},
    {"title": "Traces", "url": "https://jaeger.example.com/search?service=order-service&tags=%7B%22error%22%3A%22true%22%7D"}
  ]
}
```

### Dashboard as Code

Avoid manually creating dashboards in the UI — they become inconsistent and untracked. Use code:

```python
# Grafonnet (JSONNET) — compile to Grafana JSON
local grafana = import 'grafonnet/grafana.libsonnet';
local dashboard = grafana.dashboard;
local graphPanel = grafana.graphPanel;
local prometheus = grafana.prometheus;

dashboard.new('Order Service', uid='order-service')
.addPanel(
  graphPanel.new('Request Rate')
    .addTarget(prometheus.target(
      'rate(http_requests_total{service="order-service"}[5m])',
      legendFormat='{{method}} {{endpoint}}'
    )),
  gridPos={x: 0, y: 0, w: 12, h: 8}
)
```

Or use **Grafana provisioning** (YAML-defined dashboards loaded from disk):
```yaml
# grafana/provisioning/dashboards/order-service.yaml
apiVersion: 1
providers:
  - name: 'default'
    folder: 'Services'
    type: file
    options:
      path: /etc/grafana/dashboards
```

### The Three Pillars Connected

```
Incident workflow using all three pillars:

1. ALERT fires: Error rate > 1%
   │
   ▼
2. DASHBOARD (metrics): confirm error rate spike at 14:32
   │ "p99 latency jumped from 150ms to 2000ms at the same time"
   │
   ▼
3. TRACES: click exemplar on the latency histogram spike
   │ "Database span taking 1800ms — slow query on orders table"
   │
   ▼
4. LOGS: filter logs at 14:32 for trace_id
   "WARN: table scan on orders table — missing index on customer_id"
   │
   ▼
5. RESOLUTION: add index → latency drops → alert resolves
```

---

## Key Trade-offs

| Dimension | Grafana (self-hosted) | Managed (Datadog, New Relic) |
|---|---|---|
| Cost | Infrastructure only | Per-host / per-user pricing (expensive at scale) |
| Setup | Medium complexity | Low (SaaS) |
| Customization | Unlimited | Constrained by vendor |
| Data sovereignty | Full control | Data leaves your infra |
| Features | Community plugins | Integrated ML anomaly detection, APM |

---

## When to use

- Build a service health dashboard for every production service before launch.
- SLO dashboards should be visible to the entire team — not just on-call. Use them in incident reviews and planning.
- Use dashboard-as-code from the start to avoid dashboard sprawl and enable version control.

---

## Common Pitfalls

- **Dashboard sprawl**: hundreds of dashboards, most stale and unowned. Establish a governance process — link dashboards to owning teams, audit and delete unused ones quarterly.
- **Using averages instead of percentiles**: a dashboard showing average latency=50ms looks healthy when p99=5000ms. Always show p50, p95, p99 in latency panels.
- **Not linking to runbooks from the dashboard**: when an alert fires and the engineer opens the dashboard, the runbook link should be one click away — in the panel description or annotation.
- **Dashboards without variable templating**: hardcoding `instance="prod-pod-1"` means creating a new dashboard for every new service instance. Use Grafana template variables for environment, service, and instance.
- **Ignoring the long-tail time window**: dashboards default to the last 1 hour. Add a "30-day" time range shortcut for SLO dashboards — monthly budget visibility is critical for planning escalation decisions.

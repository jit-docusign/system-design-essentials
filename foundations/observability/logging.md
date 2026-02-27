# Logging

## What it is

**Logging** is the practice of recording discrete events that occur in a system — what happened, when, and in what context. Logs are the primary tool for **post-hoc debugging**: investigating what went wrong after an incident.

The modern approach is **structured logging** — emitting logs as machine-parseable key-value records (JSON) rather than freeform strings — which enables fast searching, filtering, and aggregation across millions of log lines.

---

## How it works

### Log Levels

| Level | Meaning | When to use |
|---|---|---|
| `TRACE` | Finest-grained execution detail | Function entry/exit, loop iterations (dev only) |
| `DEBUG` | Diagnostic info for developers | Variable values, branch decisions |
| `INFO` | Normal operational events | Request received, order placed, user logged in |
| `WARN` | Something unexpected but handled | Retry #2 of 3, deprecated API used, approaching threshold |
| `ERROR` | Operation failed, needs attention | DB query failed, external API returned 500 |
| `FATAL` / `CRITICAL` | System cannot continue | Cannot connect to database on startup |

**Production rule**: default to INFO. Enable DEBUG only temporarily via a feature flag or log sampling — DEBUG logs can 10x your log volume and cost.

### Structured vs Unstructured Logging

```
# Unstructured (avoid in production):
logger.info(f"User {user_id} placed order {order_id} for {amount} USD at {timestamp}")

# Output:
2024-01-15 14:23:01 INFO User 8721 placed order 9981 for 49.99 USD at 2024-01-15T14:23:01Z

# Hard to query: "Find all orders > $100" requires regex parsing.
```

```python
# Structured (JSON) logging with structlog:
import structlog
import logging

structlog.configure(
    processors=[
        structlog.contextvars.merge_contextvars,
        structlog.processors.add_log_level,
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.dev.ConsoleRenderer() if DEV else structlog.processors.JSONRenderer(),
    ]
)

log = structlog.get_logger()

log.info("order_placed",
    user_id="8721",
    order_id="9981",
    amount_usd=49.99,
    currency="USD",
    items_count=3,
    payment_method="card",
)
```

Output:
```json
{
  "event": "order_placed",
  "level": "info",
  "timestamp": "2024-01-15T14:23:01.234Z",
  "user_id": "8721",
  "order_id": "9981",
  "amount_usd": 49.99,
  "currency": "USD",
  "items_count": 3,
  "payment_method": "card",
  "service": "order-service",
  "version": "2.4.1",
  "trace_id": "abc123def456"  // Add trace ID for correlation
}
```

### Contextual Logging with Request Scoped Fields

Attach fields once and have them automatically included in all subsequent log calls within that request:

```python
import structlog
from contextvars import ContextVar

log = structlog.get_logger()

# Middleware: set request-scoped context
async def logging_middleware(request, call_next):
    structlog.contextvars.clear_contextvars()
    structlog.contextvars.bind_contextvars(
        request_id=request.headers.get("X-Request-ID"),
        trace_id=request.headers.get("X-Trace-ID"),
        user_id=request.state.user_id if hasattr(request.state, "user_id") else None,
        method=request.method,
        path=request.url.path,
    )
    try:
        response = await call_next(request)
        log.info("request_completed", status_code=response.status_code)
        return response
    except Exception as e:
        log.error("request_failed", error=str(e), exc_info=True)
        raise

# Anywhere deep in the call stack, trace_id is automatically included:
def process_payment(order_id: str, amount: float):
    log.info("processing_payment", order_id=order_id, amount=amount)
    # Output automatically includes trace_id, request_id, user_id from context
```

### Correlation IDs

Link logs across services using a propagated trace/correlation ID:

```
Request starts in API Gateway → generates correlation_id = "abc-123"
  │
  ├── Order Service  logs with trace_id="abc-123"
  │      │
  │      ├── Payment Service logs with trace_id="abc-123"
  │      │
  │      └── Inventory Service logs with trace_id="abc-123"

Query in log aggregator:
  trace_id="abc-123" → shows complete request journey across all services
```

Pass via HTTP headers:
```python
# Outbound call: propagate the trace ID
headers = {
    "X-Trace-ID": current_trace_id(),
    "X-Request-ID": str(uuid.uuid4()),
}
response = httpx.get("http://payment-service/charge", headers=headers)
```

### Log Aggregation Architecture

```
Services (K8s pods)     Log Collector     Storage/Search     UI
┌──────────────┐        ┌──────────┐     ┌────────────────┐  ┌──────────┐
│ Order Svc    │──JSON──►          │     │                │  │          │
│ logs to stdout│        │ Fluent   │────►│ Elasticsearch  │──►│  Kibana  │
├──────────────┤        │ Bit /    │     │   or Loki      │  │  Grafana │
│ Payment Svc  │──JSON──► Logstash /│    │                │  │          │
├──────────────┤        │ Vector   │     └────────────────┘  └──────────┘
│ Inventory Svc│──JSON──►          │
└──────────────┘        └──────────┘

Output retention policy:
  DEBUG: 1 day    (high volume, low value)
  INFO:  7 days
  WARN:  30 days
  ERROR: 90 days
  AUDIT: 1–7 years (compliance)
```

### ELK Stack vs Grafana Loki

| Dimension | ELK (Elastic) | Grafana Loki |
|---|---|---|
| Indexing | Full-text index on all fields | Index only labels (stream metadata) |
| Cost | High storage (indexes ~2x raw) | Low storage (only raw logs + small index) |
| Query speed | Fast on any field | Fast on label queries; full-text search slower |
| Complexity | High (Elasticsearch tuning) | Low (similar to Prometheus) |
| Best for | Complex log analytics, compliance | Cloud-native with Prometheus + Grafana |

```
# Loki query examples (LogQL):
{namespace="production", app="order-service"} |= "error"
{app="payment"} | json | amount_usd > 1000
rate({app="api-gateway"}[5m])  — log lines per second
```

### Sampling

High-throughput services can emit millions of log lines per second. Use **log sampling** to control volume:

```python
import random

log = structlog.get_logger()

def process_request(request):
    # Log all errors and 1% of successful requests
    result = handle(request)
    
    if result.is_error:
        log.error("request_error", path=request.path, error=result.error)
    elif random.random() < 0.01:  # 1% sampling for INFO
        log.info("request_processed", path=request.path, duration_ms=result.duration_ms)
```

---

## Key Trade-offs

| Dimension | Structured JSON logs | Unstructured text logs |
|---|---|---|
| Queryability | Rich filtering on any field | Regex only |
| Human readability | Harder at a glance | Easy to read directly |
| Storage | Slightly larger per line | Smaller |
| Integration | First-class in ELK/Loki | Requires parsing rules |

| Dimension | Centralized aggregation | Per-service local logs |
|---|---|---|
| Correlation across services | Easy | Manual / impossible |
| Cost | Higher (storage, transfer) | Zero |
| Operational dependency | High (if aggregator goes down) | None |

---

## When to use

- Logs are the right tool for **discrete events** (order placed, payment failed, user signed up). For continuous measurements (request rate, CPU), use **metrics**. For request traces, use **distributed tracing**.
- Use structured JSON logging in all production services.
- Add correlation IDs from day one — retrofitting them later requires touching every service.
- Set log retention policies based on compliance requirements (GDPR: minimum necessary; PCI-DSS: 1 year; HIPAA: 6 years).

---

## Common Pitfalls

- **Logging sensitive data**: never log passwords, tokens, credit card numbers, SSNs, or health data. Use field masking or redaction in the logging pipeline. Audit logs regularly against a PII schema.
- **Printf-style logging in hot paths**: `logger.debug(f"Processing item {item}")` formats the string even if DEBUG is disabled. Use lazy evaluation: `logger.debug("Processing item", item=item)`.
- **Catching and swallowing exceptions without logging**: `except Exception: pass` makes bugs invisible. At minimum: `except Exception as e: log.error("op_failed", error=str(e), exc_info=True)`.
- **Not including trace IDs**: without correlation IDs, debugging a distributed system failure requires manually correlating timestamps across multiple log streams — extremely slow and error prone.
- **One log file per service on the same disk**: log files competing with application I/O causes performance degradation. Log to stdout/stderr (container standard) and let the orchestrator handle collection.

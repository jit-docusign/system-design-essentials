# Distributed Tracing

## What it is

**Distributed tracing** tracks a single request as it propagates through multiple services in a distributed system. It answers: "Where did those extra 500ms come from?" and "Which service in my 12-hop call chain failed?"

A **trace** is the full journey of one request from end to end. A trace is composed of **spans** — each span represents a named unit of work (an RPC call, a database query, a cache lookup) with a start time, duration, and metadata.

---

## How it works

### Trace Model

```
Trace ID: abc-123

Span: Frontend (root span)  0ms────────────────────450ms
  │
  ├── Span: API Gateway      5ms──────────────────440ms
  │     │
  │     ├── Span: Order Svc  10ms──────────────120ms
  │     │     │
  │     │     ├── Span: DB query  15ms──25ms
  │     │     └── Span: Cache get 26ms──30ms
  │     │
  │     ├── Span: Payment Svc  130ms─────────────380ms  ← bottleneck
  │     │     │
  │     │     ├── Span: Stripe API  135ms────────370ms  ← slow external call
  │     │     └── Span: DB insert   372ms─375ms
  │     │
  │     └── Span: Notification Svc  381ms──420ms
```

Key insight: the waterfall view shows that Payment Svc took 250ms and within it, the Stripe API call took 235ms — immediately pinpointing the bottleneck.

### Span Structure

```json
{
  "trace_id": "abc123def456abc123def456",
  "span_id": "a1b2c3d4e5f6a1b2",
  "parent_span_id": "0102030405060708",
  "operation_name": "order_service.create_order",
  "service_name": "order-service",
  "start_time_unix_nano": 1700000000123456789,
  "duration_ns": 95000000,
  "status": "OK",
  "attributes": {
    "http.method": "POST",
    "http.url": "/api/orders",
    "http.status_code": 201,
    "order.id": "order-9981",
    "order.items_count": 3,
    "db.system": "postgresql",
    "db.statement": "INSERT INTO orders ..."
  },
  "events": [
    {
      "time": 1700000000150000000,
      "name": "inventory_check_complete",
      "attributes": {"items_available": true}
    }
  ]
}
```

### Context Propagation

For a trace to span services, the trace context (trace ID + parent span ID) must be passed in request headers:

```
W3C Trace Context (standard):
  traceparent: 00-abc123def456abc123def456-a1b2c3d4e5f6a1b2-01
               version  trace_id (128-bit)  parent_span_id    flags (sampled)

B3 Propagation (Zipkin, older):
  X-B3-TraceId: abc123def456abc123def456
  X-B3-SpanId: a1b2c3d4e5f6a1b2
  X-B3-ParentSpanId: 0102030405060708
  X-B3-Sampled: 1
```

### OpenTelemetry (Industry Standard)

OpenTelemetry (OTel) is the unified observability standard providing a vendor-neutral API and SDK for traces, metrics, and logs. Replace vendor SDKs with OTel and switch backends without code changes.

```python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
from opentelemetry.instrumentation.httpx import HTTPXClientInstrumentor
from opentelemetry.instrumentation.sqlalchemy import SQLAlchemyInstrumentor

# Setup: configure provider + exporter
provider = TracerProvider()
otlp_exporter = OTLPSpanExporter(endpoint="http://otel-collector:4317")
provider.add_span_processor(BatchSpanProcessor(otlp_exporter))
trace.set_tracer_provider(provider)

# Auto-instrument libraries — no manual span creation needed for common frameworks
FastAPIInstrumentor.instrument(app=app)       # All HTTP endpoints
HTTPXClientInstrumentor().instrument()         # All outbound HTTP calls
SQLAlchemyInstrumentor().instrument(engine=engine) # All DB queries

# Manual span for custom operations
tracer = trace.get_tracer("order-service")

async def create_order(order_data: dict) -> dict:
    with tracer.start_as_current_span("create_order") as span:
        span.set_attribute("order.customer_id", order_data["customer_id"])
        span.set_attribute("order.items_count", len(order_data["items"]))
        
        with tracer.start_as_current_span("validate_inventory"):
            for item_id in order_data["items"]:
                await check_inventory(item_id)
        
        with tracer.start_as_current_span("process_payment") as payment_span:
            payment_result = await charge_card(order_data["payment"])
            payment_span.set_attribute("payment.method", "card")
            payment_span.set_attribute("payment.amount_usd", order_data["total"])
        
        if payment_result.failed:
            span.set_status(trace.StatusCode.ERROR, "Payment failed")
            span.record_exception(payment_result.error)
        
        return {"order_id": str(uuid.uuid4())}
```

### OTel Collector (Centralized Pipeline)

The OTel Collector decouples instrumentation from backends:

```
Services
┌─────────────────────┐
│ Order Service       │──► OTLP/gRPC ──►┐
├─────────────────────┤                  │  ┌──────────────────────┐
│ Payment Service     │──► OTLP/gRPC ───►├──►                      │──► Jaeger
├─────────────────────┤                  │  │  OTel Collector      │──► Tempo
│ Frontend            │──► OTLP/HTTP ───►┘  │  (Receive,Process,  │──► Datadog
└─────────────────────┘                     │   Export)            │──► Honeycomb
                                            └──────────────────────┘
```

```yaml
# otel-collector-config.yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:
    timeout: 5s
  tail_sampling:
    decision_wait: 10s
    policies:
      - name: errors-policy
        type: status_code
        status_code: {status_codes: [ERROR]}
      - name: slow-traces-policy
        type: latency
        latency: {threshold_ms: 500}
      - name: probabilistic
        type: probabilistic
        probabilistic: {sampling_percentage: 5}

exporters:
  jaeger:
    endpoint: "jaeger:14250"
  prometheusremotewrite:
    endpoint: "http://prometheus:9090/api/v1/write"

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch, tail_sampling]
      exporters: [jaeger]
```

### Sampling Strategies

Tracing every request would generate enormous data. Sampling controls which traces are recorded:

| Strategy | Decision point | Keeps | Use case |
|---|---|---|---|
| **Head sampling (probabilistic)** | At trace start | Random N% | Low overhead, simple |
| **Tail sampling** | After span data collected | All errors + slow + N% of normal | Best coverage, higher memory |
| **Rate limiting** | Per-service | N traces/sec regardless of traffic | Predictable cost |

**Best practice**: 1–5% head sampling by default. Use tail sampling to guarantee 100% of error traces and 100% of traces exceeding your SLO threshold are captured.

### Jaeger vs Tempo vs Zipkin

| Dimension | Jaeger | Grafana Tempo | Zipkin |
|---|---|---|---|
| Storage backends | Cassandra, Elasticsearch, in-memory | Object storage (S3/GCS), local | MySQL, Cassandra, Elasticsearch |
| Grafana integration | Native datasource | Native (built for Grafana) | via Grafana plugin |
| Query capabilities | Rich UI, dependency graph | Basic (correlate with logs/metrics) | Basic UI |
| Scalability | High (distributed collectors) | Very high (write-heavy, cheap storage) | Medium |
| OTel native | Yes | Yes | Yes |

---

## Key Trade-offs

| Choice | Pros | Cons |
|---|---|---|
| **Head sampling (5%)** | Low overhead, simple | Misses rare errors |
| **Tail sampling (100% errors)** | Guaranteed error coverage | High memory in collector during decision window |
| **OTel auto-instrumentation** | Zero code changes, fast rollout | Less granular spans for custom operations |
| **Manual spans** | Precise, business-relevant context | Developer effort to add and maintain |

---

## When to use

- Use distributed tracing when debugging latency issues or failures in a **multi-service** request path. For single-service latency, profiling tools are more appropriate.
- Add tracing from the start of a microservices project — retrofitting trace context propagation across dozens of services is painful.
- Correlate traces with logs (via trace_id in log fields) and metrics (via exemplars in Prometheus histograms) for full observability coverage.

---

## Common Pitfalls

- **Not propagating trace context in async jobs**: messages sent to a Kafka queue or an async worker break the trace unless you explicitly carry the trace context into the message payload and restore it in the consumer.
- **Over-sampling in high-traffic systems**: 100% sampling at 10,000 RPS generates 10,000 traces/sec and can cost thousands of dollars per month in storage. Use 1–5% probabilistic + tail sampling for errors.
- **Forgetting error status on spans**: a span with `status=OK` that has an error attribute doesn't surface in error queries. Always call `span.set_status(StatusCode.ERROR)` on failure — this is what alerting rules query on.
- **Treating trace_id as a correlation ID in logs**: trace_id is for tracing backends, not log search. Add both `trace_id` and `request_id` to log context; use request_id for log correlation and trace_id to link to the tracing UI.

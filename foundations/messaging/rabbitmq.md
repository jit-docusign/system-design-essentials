# RabbitMQ

## What it is

**RabbitMQ** is an open-source **message broker** that implements the **AMQP** (Advanced Message Queuing Protocol). It uses a **push-based model** where the broker delivers messages to consumers, and routes messages through a flexible **exchange → queue → consumer** architecture with rich routing rules.

Unlike Kafka's log-based model where messages are retained for replay, RabbitMQ's queues **delete messages once acknowledged** — it is designed for **transient task delivery**, not event stream replay.

---

## How it works

### Core Components

```
                    ┌────── RabbitMQ Broker ──────────────────────┐
                    │                                             │
Producer ──────────►│  Exchange ──── Binding ──► Queue ──────────►│──── Consumer
  publish(          │                                             │     receive()
    exchange,       │   [routing rules]    [message buffer]       │     ack()
    routing_key,    │                                             │
    message)        └─────────────────────────────────────────────┘
```

**Exchange**: receives messages from producers and routes them to queues based on routing rules and bindings.

**Queue**: a durable buffer that stores messages until a consumer ACKs them.

**Binding**: a rule linking an exchange to a queue, with an optional routing key or pattern.

**Message**: contains a body (payload), properties (headers, priority, expiry), and routing metadata.

### Exchange Types

| Exchange Type | Routing Logic | Use Case |
|---|---|---|
| **Direct** | Exact match on routing key | Task queues (e.g., `key="email"`) |
| **Fanout** | Sends to ALL bound queues, ignores routing key | Broadcast to multiple services |
| **Topic** | Wildcard match: `*` = one word, `#` = zero or more words | Flexible routing (e.g., `logs.*.error`) |
| **Headers** | Routes based on message header attributes (not routing key) | Complex attribute-based routing |

**Direct exchange example:**
```
Exchange: order-exchange (direct)
  Binding: routing_key="inventory" → Queue: inventory-queue
  Binding: routing_key="billing"   → Queue: billing-queue
  Binding: routing_key="shipping"  → Queue: shipping-queue

Producer sends: exchange="order-exchange", routing_key="billing"
→ Only billing-queue receives the message
```

**Topic exchange example:**
```
Exchange: logs-exchange (topic)
  Binding: "*.critical"   → Queue: critical-alerts (all critical logs)
  Binding: "payments.#"   → Queue: payments-log   (all payments/* logs)
  Binding: "#"            → Queue: all-logs        (every log)

Message with routing_key="payments.critical":
→ Delivered to: critical-alerts, payments-log, all-logs
```

**Fanout exchange example:**
```
Exchange: notifications-fanout (fanout)
  Bound: email-queue
  Bound: sms-queue
  Bound: push-notification-queue

Producer sends one message → all three queues receive a copy
```

### Message Acknowledgment

**Consumer ACK model** (push-based):
```
Broker pushes message to consumer
Consumer processes: success → ack()  → broker deletes message
Consumer processes: failure → nack() → broker re-queues or moves to DLQ
Consumer crashes    → no ack       → broker re-delivers after timeout
```

**Publisher confirms** (producer ACK):
```python
channel.confirm_delivery()  # enable confirm mode

channel.basic_publish(exchange='orders', routing_key='billing', body=msg)
if channel.is_message_confirmed():
    # guaranteed to be in the queue
    pass
```

**Prefetch** (QoS): limit the number of unacknowledged messages a consumer holds at once. Prevents one consumer from hoarding all messages.
```python
channel.basic_qos(prefetch_count=10)  # max 10 unacknowledged at a time
```

### Dead Letter Exchange (DLX)

Messages are moved to a Dead Letter Exchange (DLX) when:
- They are `nack`'d without requeue.
- Their TTL expires.
- The queue length limit is exceeded.

```python
# Declare a queue with a DLX
channel.queue_declare(queue='orders', arguments={
    'x-dead-letter-exchange': 'dlx-exchange',
    'x-dead-letter-routing-key': 'failed-orders',
    'x-message-ttl': 30000,  # message expires after 30s
    'x-max-length': 10000    # max queue depth
})
```

DLX-based **delayed retry queue** — a common pattern:
```
Consumer nacks → goes to DLX
DLX routes to "retry-queue" with TTL=5min
After 5 min, TTL expires → dead-lettered back to original queue
→ Consumer retries
```

### Full Producer/Consumer Example

```python
import pika
import json

# Setup
connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

# Declare durable queue
channel.queue_declare(queue='email-jobs', durable=True)

# PRODUCER
def send_email_job(recipient, subject, body):
    channel.basic_publish(
        exchange='',
        routing_key='email-jobs',
        body=json.dumps({'to': recipient, 'subject': subject, 'body': body}),
        properties=pika.BasicProperties(
            delivery_mode=2,  # make message persistent
            priority=5
        )
    )

# CONSUMER
def on_message(ch, method, properties, body):
    job = json.loads(body)
    try:
        send_email(job['to'], job['subject'], job['body'])
        ch.basic_ack(delivery_tag=method.delivery_tag)
    except Exception as e:
        # nack with requeue=False → goes to DLX
        ch.basic_nack(delivery_tag=method.delivery_tag, requeue=False)

channel.basic_qos(prefetch_count=5)
channel.basic_consume(queue='email-jobs', on_message_callback=on_message)
channel.start_consuming()
```

### Clustering and High Availability

**Classic mirrored queues** (deprecated): queues replicated to all cluster nodes.

**Quorum queues** (recommended, v3.8+): Raft-based replicated queues. Strongly consistent. Tolerate N/2-1 broker failures.

```python
channel.queue_declare(queue='orders', arguments={
    'x-queue-type': 'quorum',
    'x-quorum-initial-group-size': 3  # replicate across 3 brokers
})
```

**Streams** (v3.9+): Kafka-like persistent, replayable log. Consumers can seek to any point. Intended for high-throughput event-stream use cases where you previously would have used Kafka.

---

## Key Trade-offs

| Advantage | Disadvantage |
|---|---|
| Rich routing (topic/fanout/headers/direct) | Messages deleted on ACK — no replay by default |
| Per-message TTL, priority, and DLX | Not designed for extremely high throughput (vs Kafka) |
| AMQP standard protocol — many client libraries | Complex broker topology to reason about at scale |
| Low latency push delivery | No consumer groups like Kafka — each queue has its own consumers |
| Mature, battle-tested, easy to reason about | Clustering and HA configuration can be tricky |
| Fine-grained consumer QoS and prefetch | Message ordering not guaranteed across multiple consumers |

---

## When to use

RabbitMQ fits well for:
- **Task/work queues**: background jobs with retry, priority, and DLQ (email sending, report generation, image processing).
- **Complex routing needs**: events need to be routed to different queues based on attributes (headers, routing keys, wildcards).
- **Per-message TTL / priority queues**: jobs with deadlines or urgency levels.
- **When replay is not needed**: one-time task delivery where "delete on ACK" is correct.
- **Smaller-scale messaging**: < a few hundred thousand messages/second.

Prefer Kafka when:
- You need consumer groups and message replay.
- Throughput is millions of events/second.
- You're building an event sourcing or CDC pipeline.

---

## Common Pitfalls

- **Non-durable queues and non-persistent messages**: by default, queues are not durable and messages are not persistent. A broker restart loses everything. Always set `durable=True` and `delivery_mode=2` for production.

- **Not setting prefetch count**: consumers receive unlimited unacknowledged messages by default, leading to memory bloat on the consumer and starvation of other consumers. Always set `basic_qos(prefetch_count=N)`.

- **Requeuing indefinitely**: `nack(requeue=True)` on a processing error puts the message back at the head of the queue, creating a busy loop. Use a DLQ with a retry delay instead.

- **Not monitoring queue depth and DLQ**: rising queue depth indicates consumer lag or failures. A growing DLQ means messages are consistently failing — alert on both.

- **Classic mirrored queues in new deployments**: classic mirrors are deprecated. Use Quorum Queues for HA.

- **Forgetting to bind the exchange**: a message published to an exchange with no binding silently vanishes. Always verify bindings in development.

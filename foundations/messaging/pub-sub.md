# Publish-Subscribe (Pub/Sub)

## What it is

**Publish-Subscribe (Pub/Sub)** is a messaging pattern where **publishers** send messages to a **topic** without knowledge of who will receive them, and **subscribers** express interest in topics and receive all matching messages — without knowledge of who sent them.

Publishers and subscribers are **completely decoupled**: a publisher doesn't know how many subscribers exist, and a subscriber doesn't know which publisher sent a message. This is the foundational pattern enabling event-driven fan-out.

Common implementations: Google Cloud Pub/Sub, Amazon SNS, Apache Kafka (consumer groups), Redis Pub/Sub, NATS.

---

## How it works

### Core Pub/Sub Model

```
                         Pub/Sub Broker

Publisher A ──► Topic: "order-events" ──► Subscription A ──► Consumer: Analytics
Publisher B ──►                      ──► Subscription B ──► Consumer: Inventory
                                     ──► Subscription C ──► Consumer: Notifications
```

- Each **Subscription** is an independent copy of the message stream.
- A message published to a topic is **delivered to every subscription** — this is fan-out.
- Each subscription can have multiple consumer instances, sharing delivery load within that subscription.

This differs from a work queue: with Pub/Sub, two subscriptions each get every message; with a queue, each message goes to exactly one consumer.

### Fan-out Pattern

```
Order Service publishes:
  Topic "orders" ← { event_type: "ORDER_PLACED", order_id: "123", user_id: "456" }

Fan-out to all subscribers:
  ├── Analytics Service (builds dashboards)
  ├── Inventory Service (reserve stock)
  ├── Notification Service (send email)
  └── Fraud Detection Service (check for anomalies)

Each subscriber processes independently and at its own pace.
```

### Delivery Semantics

| Mode | Behavior |
|---|---|
| **At-most-once** | Fire-and-forget. May lose messages. Low latency. |
| **At-least-once** | Retries until ACK. Possible duplicates. Most pub/sub systems default to this. |
| **Exactly-once** | Deduplication or transactional. Complex. See [Exactly-Once Delivery](exactly-once.md). |

For **at-least-once** pub/sub, always design idempotent consumers — processing the same message twice must be safe.

```python
def handle_order_placed(event):
    order_id = event['order_id']
    # idempotent: INSERT OR IGNORE prevents duplicate processing
    db.execute("INSERT OR IGNORE INTO inventory_reservations (order_id) VALUES (?)", order_id)
```

### Pull vs Push Delivery

**Push**: the broker calls a subscriber's HTTP endpoint when a message arrives.

```
Broker ──POST /webhooks/order-events──► Subscriber Service
Subscriber processes → returns 200 (ACK) or 4xx/5xx (NACK + retry)
```

Pros: low latency, no polling overhead.  
Cons: subscriber must expose a public HTTPS endpoint; slower subscribers cause timeouts.

**Pull**: subscribers poll the broker for messages at their own pace.

```python
# Google Cloud Pub/Sub pull
from google.cloud import pubsub_v1

subscriber = pubsub_v1.SubscriberClient()
subscription_path = subscriber.subscription_path(PROJECT_ID, SUBSCRIPTION_ID)

response = subscriber.pull(request={"subscription": subscription_path, "max_messages": 10})
for msg in response.received_messages:
    process(msg.message.data)
    subscriber.acknowledge(subscription=subscription_path,
                           ack_ids=[msg.ack_id])
```

Pros: consumers control their own rate; easier to implement backpressure.  
Cons: polling adds latency; idle polling wastes resources.

### SNS + SQS Fan-out Pattern

A common AWS pattern: SNS for fan-out, SQS queues for per-consumer durability.

```
Publisher ──► SNS Topic "orders" ──► SQS Queue: analytics-queue ──► Analytics Consumer
                                 ──► SQS Queue: inventory-queue ──► Inventory Consumer
                                 ──► SQS Queue: notify-queue    ──► Notification Consumer
                                 ──► HTTP endpoint              ──► Webhook Consumer
```

Benefits:
- SNS handles fan-out.
- SQS provides durable buffering + retry per consumer independently.
- Each consumer's SQS queue can have its own DLQ.
- Consumers can autoscale independently.

### Redis Pub/Sub

Redis Pub/Sub is fire-and-forget — **no durability, no persistence**. A subscriber that's offline misses messages.

```python
# Publisher
import redis
r = redis.Redis()
r.publish('order-events', json.dumps({'type': 'ORDER_PLACED', 'order_id': '123'}))

# Subscriber
p = r.pubsub()
p.subscribe('order-events')
for message in p.listen():
    if message['type'] == 'message':
        handle(json.loads(message['data']))
```

Use Redis Pub/Sub only for ephemeral real-time fan-out (e.g., live chat, live notifications) where message loss is acceptable.

For durable pub/sub in Redis, use **Redis Streams** (which supports consumer groups and persistent log semantics).

### Filtering and Topic Design

**By topic granularity**:
```
Option A: one topic per event type
  topics: "orders.placed", "orders.shipped", "orders.cancelled"
  Subscriber can subscribe to exactly the events it needs.

Option B: one topic for all order events
  topic: "orders"
  Messages have an event_type field.
  Subscriber must filter on event_type in-process (or use subscription filters).
```

**Subscription message filters** (GCP/SNS support):

```python
# SNS filter policy: only deliver ORDER_PLACED events to inventory subscription
filter_policy = '{"event_type": ["ORDER_PLACED"]}'
sns.create_subscription(
    TopicArn=topic_arn,
    Protocol='sqs',
    Endpoint=sqs_queue_arn,
    Attributes={'FilterPolicy': filter_policy}
)
```

---

## Key Trade-offs

| Advantage | Disadvantage |
|---|---|
| Complete producer/consumer decoupling | At-least-once requires idempotent consumers |
| Scales to arbitrary number of subscribers | Message ordering generally not guaranteed |
| Independent consumer scaling and evolution | Fan-out amplifies load — 10 subscribers = 10× message processing |
| Natural audit trail when messages are retained | Schema evolution requires versioning discipline |
| Enables temporal decoupling and buffering | Subscriber lag needs active monitoring |

---

## When to use

- **Fan-out to multiple services**: one event triggers N independent reactions — analytics, inventory, notifications, fraud detection.
- **Decoupled microservice communication**: services need to be unaware of each other.
- **Audit logging**: retain every published event for compliance or debugging.
- **Cross-team integration**: team A publishes to a topic; teams B, C, D subscribe without coordinating with A.
- **Real-time notifications**: push data to browser/mobile clients via WebSocket or SSE backed by pub/sub.

Not appropriate when:
- A message must be processed by exactly one consumer (use a work queue).
- You need complex routing (use RabbitMQ topic exchanges).
- You need real-time response to the publisher (use gRPC/REST).

---

## Common Pitfalls

- **Not configuring subscription-level DLQ**: if a subscriber fails consistently, messages pile up and eventually expire. Always configure a dead-letter topic or DLQ per subscription.
- **Ignoring fan-out amplification**: 1,000 messages published × 10 subscribers = 10,000 message deliveries. Size downstream consumers accordingly.
- **Using Redis Pub/Sub for durable messaging**: Redis Pub/Sub is drop-on-disconnect. Use Redis Streams or Kafka for durability.
- **Missing subscriber lag monitoring**: a subscriber that can't keep up will either back up the subscription queue or drop messages when limits are hit.
- **All subscribers in the same subscription**: two consumer instances in the same subscription share messages (work queue). Two separate subscriptions each get their own copy of all messages. Know which model you need.

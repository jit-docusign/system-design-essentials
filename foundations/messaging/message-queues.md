# Message Queues

## What it is

A **message queue** is a form of asynchronous service-to-service communication where a **producer** sends messages to a queue, and one or more **consumers** process those messages — independently and at their own pace. The producer doesn't wait for the consumer to process the message; it returns immediately after enqueueing.

Message queues decouple producers from consumers in both time (the consumer doesn't have to be running when the message is sent) and scale (consumers can process at a different rate than producers produce).

**Common systems**: Amazon SQS, RabbitMQ, Redis (as a queue), ActiveMQ, Azure Service Bus.

---

## How it works

### Core Model

```
Producer                Queue                  Consumer
  │                      │                        │
  │  send(message)       │                        │
  ├─────────────────────►│                        │
  │                      │  receive / poll        │
  │                      │◄───────────────────────┤
  │                      │                        │
  │                      │  message               │
  │                      ├───────────────────────►│
  │                      │                        │  process()
  │                      │  ack (success)         │
  │                      │◄───────────────────────┤
  │                      │ (message deleted)      │
```

1. **Producer** sends a message to the queue.
2. **Consumer** polls (or is pushed) a message.
3. **Consumer processes** the message.
4. **Consumer acknowledges** (ACK) — the queue deletes the message.
5. If processing fails (no ACK or explicit NACK), the message returns to the queue for retry.

### Delivery Guarantees

| Guarantee | Meaning |
|---|---|
| **At-most-once** | Message is sent but may be lost. No retry. (fire-and-forget) |
| **At-least-once** | Message is retried until ACK. May be delivered more than once. |
| **Exactly-once** | Message delivered precisely once. Complex to implement. |

Most message queues default to **at-least-once**. Design consumers to be **idempotent** — processing the same message twice must have the same effect as processing it once.

```python
# Idempotent consumer: safe to process the same message multiple times
def process_payment(payment_id, amount):
    # Check if already processed
    if payment_store.exists(payment_id):
        return  # already done, skip

    # Process
    charge_card(amount)
    payment_store.record(payment_id)
```

### Acknowledgment and Retry

**Visibility timeout** (SQS model): when a consumer receives a message, it's temporarily hidden from other consumers. If the consumer doesn't ACK within the visibility timeout, the message reappears for another consumer to try.

```
Consumer receives message → message hidden for 30s
Consumer processes in 5s → ACK → message deleted
Consumer crashes at 10s → message reappears after 30s → another consumer processes it
```

**Dead Letter Queue (DLQ)**: messages that fail after N retries are moved to a DLQ for inspection and manual intervention. Prevents poison messages from blocking the queue indefinitely.

```
Message fails 3 times → moved to DLQ
Operations team inspects DLQ → fixes the bug → replays messages
```

### Queue Patterns

**Point-to-Point (Work Queue):**
Messages are distributed across multiple consumers — each message is processed by exactly one consumer. Used for **task distribution and load balancing**.

```
Producer ──► Queue ──► Consumer 1 (processes job 1)
                  ──► Consumer 2 (processes job 2)
                  ──► Consumer 3 (processes job 3)
```

Use case: image processing pipeline, email sending worker, order fulfillment jobs.

**Fan-out (Publish-Subscribe):**
Each message is delivered to all subscribers. Multiple consumers each get a copy.
See [Pub/Sub](pub-sub.md) for the full pattern.

**Priority Queue:**
Messages have priority levels. High-priority messages are consumed before low-priority ones, even if enqueued later.

```
Queue: [Priority-1: urgent-notification] [Priority-5: analytics-event]
       → consumer gets urgent-notification first
```

**Delayed Queue:**
Messages become visible only after a specified delay. Used for scheduling future work:
```
send_email.delay(user_id, delay_seconds=3600)  # email in 1 hour
```

### Backpressure

If consumers are slower than producers, the queue depth grows:
```
Producer: 1,000 messages/sec
Consumer: 800 messages/sec  → queue grows 200 messages/sec

After 10 min: queue has 120,000 messages (10min × 60s × 200)
```

**Backpressure** mechanisms:
1. **Scale consumers horizontally**: add more consumer instances.
2. **Rate-limit producers**: producers check queue depth and slow down.
3. **Reject excess messages**: producer gets a rejection error when queue is full.
4. **Alert on queue depth**: SLA on processing lag triggers operational response.

---

## Amazon SQS Specifics

**Standard queue**: high throughput, at-least-once delivery, best-effort ordering.  
**FIFO queue**: exactly-once processing, strict message ordering, lower throughput (300 msg/s, or 3,000 with batching).

```python
import boto3

sqs = boto3.client('sqs')

# Send
sqs.send_message(
    QueueUrl='https://sqs.us-east-1.amazonaws.com/12345/my-queue',
    MessageBody=json.dumps({'job_type': 'resize_image', 'image_id': 'img-001'}),
    MessageAttributes={
        'Priority': {'StringValue': 'high', 'DataType': 'String'}
    }
)

# Receive + Process + Delete (ACK)
response = sqs.receive_message(QueueUrl=QUEUE_URL, MaxNumberOfMessages=10,
                               VisibilityTimeout=30)
for msg in response.get('Messages', []):
    process(json.loads(msg['Body']))
    sqs.delete_message(QueueUrl=QUEUE_URL, ReceiptHandle=msg['ReceiptHandle'])
```

---

## Key Trade-offs

| Advantage | Disadvantage |
|---|---|
| Decouples producers from consumers | Adds latency (async — not real-time) |
| Absorbs traffic spikes (buffering) | Eventual processing; can't immediately confirm completion |
| Horizontal consumer scaling | At-least-once requires idempotent consumers |
| Fault tolerance — messages persist until ACK | Monitoring queue depth and DLQ is required |
| Independent scaling and deployment | Ordering not guaranteed in standard queues |

---

## When to use

Message queues are appropriate when:
- **The producer and consumer should scale independently**: email sending shouldn't block order processing.
- **The task can be processed asynchronously**: report generation, video encoding, notifications.
- **You need to buffer traffic spikes**: a Black Friday flash sale spikes orders; a queue buffers them for steady processing.
- **You need fan-out**: one event triggers multiple downstream processes.
- **Retry logic needs to be durable**: failed tasks must survive application restart.

---

## Common Pitfalls

- **Not handling idempotency**: at-least-once delivery means double-processing will happen. Design every consumer to be safe to run twice on the same message.
- **Not configuring Dead Letter Queues**: without DLQ, a bad message (poison pill) causes infinite retries and blocks healthy messages. Always configure DLQ and maximum retry count.
- **Ignoring queue depth**: rising queue depth is an early warning of consumer lag or bugs. Monitor and alert.
- **Using queues for synchronous operations**: if the API response must include the task result, a queue is wrong. Use sync calls or a request-response pattern with correlation IDs if latency matters.
- **Long visibility timeout without heartbeat**: if processing takes 10 minutes but visibility timeout is 30 seconds, the message reappears while it's still being processed. Either set a longer timeout or extend it programmatically.

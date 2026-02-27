# Exactly-Once Delivery

## What it is

**Exactly-once delivery** guarantees that a message is processed **precisely one time** — not zero times (at-most-once) and not more than once (at-least-once). It is the most desirable but most complex delivery semantic to achieve in distributed messaging systems.

The challenge arises from the fundamental tension in distributed systems: networks fail, nodes crash, and retries are necessary — but retries can cause double-processing. Achieving exactly-once requires coordination between the producer, the broker, and the consumer.

---

## How it works

### The Three Delivery Semantics

```
Event lost?    Event duplicated?    Description
     │               │
     ▼               ▼
At-most-once:  YES          NO      Message may be lost if broker/consumer fails
At-least-once: NO           YES     Retries ensure delivery; duplicates possible
Exactly-once:  NO           NO      The hardest to achieve; requires idempotency + coordination
```

**Why exactly-once is hard:**

```
Producer sends message
Network fails after broker receives but before ACK reaches producer
Producer cannot distinguish:
  (a) Message reached broker → retry = duplicate
  (b) Message never reached broker → retry = correct

Without idempotency, the producer must choose: risk loss or risk duplicate.
```

### Strategy 1: Idempotent Producers (Kafka)

Kafka's **idempotent producer** assigns each message a unique **Producer ID (PID)** and a monotonically increasing **sequence number** per partition. The broker deduplicates any duplicate with the same PID + sequence.

```python
producer = KafkaProducer(
    bootstrap_servers=['kafka:9092'],
    enable_idempotence=True,     # assigns PID, tracks sequence numbers
    acks='all',
    max_in_flight_requests_per_connection=5  # max 5 outstanding (Kafka default for idempotence)
)
```

This prevents duplicates within a **single producer session**. If the producer restarts, it gets a new PID — so it only protects against duplicates from retries within one producer lifetime.

### Strategy 2: Transactional Messaging (Kafka)

Kafka's **transactional API** extends idempotent producers to write atomically across multiple partitions and topics. A consumer configured with `isolation.level=read_committed` only sees messages from committed transactions.

```python
producer = KafkaProducer(
    bootstrap_servers=['kafka:9092'],
    transactional_id='order-processor-1',  # stable ID persists across restarts
    enable_idempotence=True
)

producer.init_transactions()

try:
    producer.begin_transaction()
    producer.send('order-events', key=b'order-123', value=order_payload)
    producer.send('audit-log', key=b'order-123', value=audit_payload)
    # Optionally commit offsets transactionally (for consume-transform-produce)
    producer.send_offsets_to_transaction(offsets, 'consumer-group-id')
    producer.commit_transaction()
except Exception:
    producer.abort_transaction()
```

Consumer: 
```python
consumer = KafkaConsumer(
    'order-events',
    isolation_level='read_committed'  # only see committed transaction messages
)
```

This achieves exactly-once **within Kafka** for consume-transform-produce pipelines.

### Strategy 3: Idempotent Consumers (Universal)

Even without broker-level exactly-once, you can achieve **effective exactly-once** by designing **idempotent consumers** — consumers that can safely process the same message multiple times without side effects.

**Pattern A: Natural Idempotency**
Some operations are naturally idempotent:
```python
# Setting a value is idempotent: setting it twice = same outcome
UPDATE users SET email = ? WHERE user_id = ?

# Incrementing is NOT idempotent:
UPDATE accounts SET balance = balance + 100 WHERE id = ?  # double-processing = double charge!
```

**Pattern B: Deduplication Table**
```python
def process_payment(payment_id, amount, user_id):
    conn = get_db()
    
    # Atomic check-and-insert (unique constraint on idempotency_key)
    try:
        conn.execute(
            "INSERT INTO processed_payments (payment_id, processed_at) VALUES (?, NOW())",
            payment_id
        )
    except UniqueConstraintError:
        return  # already processed, skip
    
    # Safe to process
    conn.execute("UPDATE accounts SET balance = balance - ? WHERE user_id = ?",
                 amount, user_id)
    conn.commit()
```

**Pattern C: Idempotency Key in Client APIs**
Stripe's approach: clients include a unique `Idempotency-Key` header. The server stores request → result mapping for 24h. Retried requests return the cached response.

```
POST /v1/charges  HTTP/1.1
Idempotency-Key: key_abc123

{amount: 1000, currency: "usd", customer: "cus_xyz"}

→ First call: processes charge, stores result for key_abc123
→ Retry with same key: returns cached result, no double charge
```

### Strategy 4: Transactional Outbox Pattern

Ensures the database write and the message emission are atomic — the message is always emitted if and only if the DB write succeeds.

```
Without Outbox (vulnerable):
  1. DB.write(order record)  ✓
  2. Broker.publish(event)   ✗ broker down → event never published, state inconsistent

With Outbox (safe):
  1. DB transaction:
       a. INSERT INTO orders (...)
       b. INSERT INTO outbox_events (event_type, payload, published=false)
     COMMIT  ← both succeed or both fail atomically
  
  2. Outbox poller (separate process):
       SELECT * FROM outbox_events WHERE published = false
       → publish to broker
       → UPDATE outbox_events SET published = true WHERE id = ?
```

The outbox table acts as a durable staging area. Even if the broker is down, events aren't lost — they're retried when the broker recovers. Use CDC (Debezium) to read the outbox table via WAL instead of polling to reduce latency.

### Comparison of Approaches

| Approach | Where Exactly-Once | Complexity | Use When |
|---|---|---|---|
| Idempotent producer (Kafka) | Producer → Broker | Low | Kafka; protects against producer retry duplicates |
| Transactional Kafka | Producer → Broker → Consumer | Medium | Kafka consume-transform-produce pipelines |
| Idempotent consumer (dedupe table) | Consumer processing | Medium | Universal; works with any broker |
| Transactional Outbox | DB write + message emission | Medium | Ensure events published iff DB committed |
| Idempotency Key API | API / server-side | Low | Client-facing APIs with retries (payments, orders) |

---

## Key Trade-offs

| Advantage | Disadvantage |
|---|---|
| Eliminates double-processing side effects | Higher implementation complexity |
| Correct behavior for financial, inventory, billing systems | Requires stateful coordination (dedup tables, broker support) |
| Enables safe retries anywhere in the stack | Performance overhead (transaction coordination, sequence tracking) |
| Predictable system behavior under failure | Dedup storage grows over time (needs TTL/cleanup) |

---

## When to use

Exactly-once guarantees are critical when:
- **Financial operations**: double-charging a card or crediting an account twice is a business-critical error.
- **Inventory reservation**: reserving the same item twice causes overselling.
- **Order placement**: creating duplicate orders is user-visible and expensive to reconcile.
- **Email/SMS sending**: sending the same confirmation email 3 times damages user trust.

For analytics, logging, and metrics pipelines, **at-least-once with idempotent sinks** is usually sufficient and simpler to implement.

---

## Common Pitfalls

- **Assuming the broker handles everything**: even with Kafka transactions, if consumers are not `read_committed` or side effects are non-idempotent (e.g., HTTP call to an external API), you still get duplicates.
- **TTL too short on dedup table**: if a message is retried after the dedup record expires, you get a duplicate. Use 24–48h TTL; monitor for messages retried after the window.
- **Non-idempotent side effects**: sending to an external API (email, payment gateway) is not idempotent by default. Always use the external service's idempotency key API.
- **UUID collision**: deduplication relies on globally unique IDs. Ensure message IDs are truly unique (UUID v4 or ULID), not auto-increment integers.
- **Applying exactly-once where at-least-once is fine**: adding dedup overhead to an analytics pipeline that can tolerate a 0.01% duplicate rate wastes complexity. Match the guarantee to the business requirement.

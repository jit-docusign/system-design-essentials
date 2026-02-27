# Event-Driven Architecture

## What it is

**Event-Driven Architecture (EDA)** is a design paradigm where services communicate by producing and consuming **events** — records of something that happened in the system — rather than by calling each other directly.

An **event** represents a **fact**: "Order #123 was placed", "User #456 logged in", "Payment #789 succeeded". Events are immutable and past-tense. Services that care about certain events subscribe to them and react accordingly — without the producer knowing or caring who is listening.

This is fundamentally different from **request/response** (REST, gRPC), where the caller waits for a synchronous reply.

---

## How it works

### Core Concepts

| Concept | Description |
|---|---|
| **Event** | An immutable record of something that happened. Contains event type, timestamp, payload, and correlation ID. |
| **Producer** | The service that emits an event when something happens. Has no knowledge of consumers. |
| **Consumer** | A service that subscribes to one or more event types and reacts to them. |
| **Event Broker** | The infrastructure that routes events from producers to consumers (Kafka, RabbitMQ, SNS, EventBridge). |
| **Event Schema** | The structure/contract of an event — typically JSON or Protobuf. |

### Event Flow

```
Order Service          Event Broker        Inventory Service
    │                      │                      │
    │ ORDER_PLACED event   │                      │
    ├─────────────────────►│                      │
    │                      │ ORDER_PLACED event   │
    │                      ├─────────────────────►│
    │                      │                      │ reserve_stock()
    │                      │                      │
    │                      │              Notification Service
    │                      │                      │
    │                      │ ORDER_PLACED event   │
    │                      ├─────────────────────►│
    │                      │                      │ send_confirmation_email()
```

Both Inventory and Notification services react to the same event independently. Order Service doesn't know or care — it just fires the event.

### Event Structure

```json
{
  "event_id": "evt-a7f8b4c2",
  "event_type": "order.placed",
  "aggregate_id": "order-123",
  "timestamp": "2024-01-15T10:30:00Z",
  "version": 1,
  "source": "order-service",
  "correlation_id": "req-7d9f4e12",
  "payload": {
    "order_id": "order-123",
    "user_id": "user-456",
    "items": [{"sku": "PROD-001", "qty": 2, "price": 29.99}],
    "total": 59.98
  }
}
```

Always include: event_type, timestamp, correlation_id, version (for schema evolution).

### Choreography vs Orchestration

**Choreography**: each service independently reacts to events and emits its own events. No central coordinator.

```
OrderService
  ─► emits ORDER_PLACED
       PaymentService listens ─► processes payment ─► emits PAYMENT_SUCCEEDED
           InventoryService listens ─► reserves stock ─► emits STOCK_RESERVED
               ShippingService listens ─► creates shipment ─► emits SHIPMENT_CREATED
```

Pros: Loose coupling, no single point of failure, easy to add new consumers.  
Cons: Business flow is implicit — hard to see the end-to-end workflow. Debugging distributed failures is complex.

**Orchestration**: a central "saga orchestrator" or "process manager" explicitly tells each service what to do and handles failures.

```
Orchestrator
  ─► calls PaymentService.charge()
       ─► success → calls InventoryService.reserve()
            ─► success → calls ShippingService.ship()
                 ─► failure → calls InventoryService.release()
                               calls PaymentService.refund()
```

Pros: Business logic is explicit and centralized. Easier to reason about failure recovery.  
Cons: Orchestrator is a coupling point. See [Saga Pattern](../microservices/saga-pattern.md) for full detail.

### Event Sourcing vs EDA

These are related but distinct:

| Event-Driven Architecture | Event Sourcing |
|---|---|
| Architectural style — services communicate via events | Pattern — state is stored as a log of events |
| Events broadcast to multiple consumers | Events are the only source of truth for a single aggregate |
| About service integration | About state management |
| Can exist without event sourcing | Works well with EDA but can be used in a monolith |

They are often used together, but neither requires the other.

### Delivery Guarantees

```
Producer:
  1. Write to DB (order placed)
  2. Emit event to broker  ← can fail between steps 1 and 2

Solve with Transactional Outbox Pattern:
  1. Write to DB + write event to 'outbox' table (same transaction)
  2. Outbox poller reads unprocessed events → publishes to broker → marks as sent
```

The **Transactional Outbox** pattern ensures events are always published even if the broker is temporarily unavailable.

```sql
-- In the same DB transaction:
INSERT INTO orders (id, user_id, total) VALUES (?, ?, ?);
INSERT INTO outbox (event_type, payload, created_at, sent) 
  VALUES ('order.placed', '{"order_id":...}', NOW(), false);
COMMIT;

-- Outbox poller:
SELECT * FROM outbox WHERE sent = false LIMIT 100;
-- Publish to broker, then:
UPDATE outbox SET sent = true WHERE id IN (?);
```

---

## Key Trade-offs

| Advantage | Disadvantage |
|---|---|
| Loose coupling — services evolve independently | Eventual consistency — consumers may lag behind |
| Easy to add new consumers without changing producers | Hard to trace end-to-end flows (need distributed tracing) |
| Natural fit for event sourcing and audit logs | Schema evolution requires versioning discipline |
| Better fault isolation | Testing is harder — need to simulate event flows |
| Handles traffic spikes via buffering | Debugging is more complex than request/response |
| Enables temporal decoupling | Not suitable for request/response use cases |

---

## When to use

EDA is appropriate when:
- **Multiple services need to react to the same business event**: an order placed should notify inventory, billing, shipping, and analytics independently.
- **You want to add new consumers without modifying the producer**: add a new analytics consumer without touching Order Service.
- **Services have different availability SLAs**: the email notification service can lag without blocking checkout.
- **Building audit trails or activity feeds**: events are natural records of what happened.
- **Separating write-heavy from read-heavy workloads** (CQRS): events propagate writes to read projections.

EDA is a poor fit when:
- **You need synchronous, real-time responses**: user-facing APIs that return immediate results.
- **Strong consistency is required**: financial ledger where all parties must agree instantly.
- **The system is simple**: EDA adds overhead — don't use it for two services that only ever talk to each other.

---

## Common Pitfalls

- **Ignoring schema evolution**: when you change an event schema, old consumers break if you aren't backward-compatible. Use versioning (`event_type: "order.placed.v2"`) or schema registries (Confluent Schema Registry with Avro/Protobuf).

- **Event as a command**: `SEND_EMAIL` is a command. `USER_REGISTERED` is an event. Events describe what happened; they don't tell consumers what to do. If you model events as commands, you've created coupling in disguise.

- **No correlation ID**: without a correlation ID, you cannot trace a request across multiple services and events. Always include correlation_id and propagate it through every downstream event.

- **Missing dead-letter handling**: consumer failures are silent unless you route unprocessable events to a dead-letter topic and alert on it.

- **Unbounded event payload growth**: don't put everything in the event payload. Use the "thin event" pattern: emit the event with just IDs, and let consumers fetch details they need. This avoids large payloads and keeps events durable over time.

- **Not versioning events**: events are stored forever. Without versioning, changing an event schema breaks consumers that replay old events.

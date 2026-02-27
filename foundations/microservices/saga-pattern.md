# Saga Pattern

## What it is

The **Saga pattern** is a way to maintain data consistency across multiple microservices **without using distributed transactions**. A saga is a sequence of **local transactions**, one per service. If any step fails, the saga executes **compensating transactions** to undo the work already done by previous steps.

This is the standard solution to the distributed transaction problem in microservices — where each service has its own database and a traditional two-phase commit (2PC) is impractical due to its locking, latency, and coupling requirements.

---

## How it works

### The Distributed Transaction Problem

```
Monolith (easy):
  BEGIN TRANSACTION
    deduct inventory
    charge payment
    create shipment
  COMMIT  ← atomic, all-or-nothing

Microservices (hard):
  Inventory DB ──────────── separate database
  Payment DB  ──────────── separate database
  Shipping DB ──────────── separate database
  
  No single transaction can span all three. Need a different approach.
```

### Saga Execution

A saga replaces one ACID transaction with a sequence of smaller local transactions + compensating transactions:

```
Step 1: Inventory Service  → reserve_stock(order)
  Success → Step 2
  Failure → Saga fails at Step 1 (nothing to undo)

Step 2: Payment Service    → charge_card(order)
  Success → Step 3
  Failure → compensate: cancel_reservation(order)  ← undo Step 1

Step 3: Shipping Service   → create_shipment(order)
  Success → Saga complete ✓
  Failure → compensate: refund_payment(order)       ← undo Step 2
             compensate: cancel_reservation(order)  ← undo Step 1
```

**Compensating transaction**: a business-level undo. It doesn't "roll back" in the database sense; it creates a new record that logically cancels the previous step. For example, a payment charge is compensated by a refund — not a database delete.

### Two Saga Implementations

#### Choreography (Event-Based)

Each service publishes an event after completing its local transaction. The next service listens and reacts. No central coordinator.

```
Order Service  ──► ORDER_PLACED ──► Inventory Service
   ↓                                   ↓
INVENTORY_RESERVED ◄───────────────────┘
   ↓
Payment Service ──► PAYMENT_CHARGED
   ↓
Shipping Service ──► SHIPMENT_CREATED


Failure scenario:
Payment Service fails ──► PAYMENT_FAILED
   ↓
Inventory Service listens ──► cancel_reservation()
                         ──► INVENTORY_RELEASED
```

Pros: fully decoupled, no single point of failure, easy to add steps.  
Cons: hard to trace the overall workflow; compensations are triggered reactively.

#### Orchestration (Centralized Coordinator)

A **Saga Orchestrator** (or Process Manager) explicitly tells each service what to do, tracks the saga state, and invokes compensations on failure.

```
Orchestrator:
  1. call Inventory.reserve(order)   ──► success
  2. call Payment.charge(order)      ──► FAILURE
  3. call Inventory.cancel(order)    ──► compensate Step 1
  4. Mark saga as FAILED

State machine:
  PENDING ──► INVENTORY_RESERVED ──► PAYMENT_CHARGED ──► COMPLETED
                                  ──► PAYMENT_FAILED ──► COMPENSATING ──► FAILED
```

Pros: explicit, auditable, easy to understand end-to-end flow; compensations controlled.  
Cons: orchestrator is a coupling point and must be highly available; more code.

### Full Orchestrated Saga Example

```python
# Saga orchestrator using a state machine

class PlaceOrderSaga:
    def __init__(self, order):
        self.order = order
        self.state = 'PENDING'
        self.reserved = False
        self.charged = False

    def execute(self):
        try:
            # Step 1: Reserve inventory
            inventory_service.reserve(self.order)
            self.reserved = True
            self.state = 'INVENTORY_RESERVED'

            # Step 2: Charge payment
            payment_service.charge(self.order)
            self.charged = True
            self.state = 'PAYMENT_CHARGED'

            # Step 3: Create shipment
            shipping_service.create_shipment(self.order)
            self.state = 'COMPLETED'
            return 'SUCCESS'

        except InventoryError as e:
            self.state = 'FAILED'
            return 'FAILED'  # nothing to compensate

        except PaymentError as e:
            self.state = 'COMPENSATING'
            self._compensate()
            return 'FAILED'

        except ShippingError as e:
            self.state = 'COMPENSATING'
            self._compensate()
            return 'FAILED'

    def _compensate(self):
        if self.charged:
            payment_service.refund(self.order)
        if self.reserved:
            inventory_service.cancel(self.order)
        self.state = 'COMPENSATED'
```

### Saga vs 2PC

| Aspect | Saga | 2PC (Two-Phase Commit) |
|---|---|---|
| Consistency model | Eventual consistency | Strong consistency |
| Lock duration | No long-lived locks | Locks held during prepare phase |
| Availability | High (no coordinator lock) | Coordinator failure blocks all participants |
| Failure handling | Compensating transactions | Rollback |
| Coupling | Loose (event-based choreography) or moderate (orchestrator) | Tight (distributed protocol) |
| Use case | Microservices | Single-vendor distributed databases |

### Handling Rollback Limitations

Compensating transactions have limits:
- **Irreversible actions**: an email was sent, a webhook was called — these can't be "un-sent". Design these as the final step in a saga (late-binding).
- **Compensations can fail too**: retry compensations with idempotency keys until they succeed. The saga never leaves a half-compensated state.
- **Concurrent modifications**: between saga step 1 and compensation, another process might have modified the data. Use version numbers or guard conditions in compensations.

---

## Key Trade-offs

| Advantage | Disadvantage |
|---|---|
| No distributed locking or 2PC | Eventual consistency (intermediate dirty reads possible) |
| Services remain loosely coupled | Compensating transactions add code complexity |
| High availability — no blocking coordinator | Hard to reason about all failure/compensation paths |
| Works across heterogeneous databases | Saga state must be persisted to survive orchestrator crash |
| Natural audit trail of saga events | Choreography: emergent behavior is hard to trace |

---

## When to use

Use the Saga pattern when:
- A **business operation spans multiple microservices** with independent databases.
- **Eventual consistency is acceptable** (the order fulfillment process can tolerate a short window of inconsistency).
- You want to avoid the complexity and coupling of distributed transactions (XA/2PC).

Not needed when:
- The operation touches a single service/database (use local ACID transactions).
- You're working in a monolith.
- Strong consistency is non-negotiable (e.g., bank transfers require ACID or XA).

---

## Common Pitfalls

- **No idempotency on saga steps**: the orchestrator may retry a step after a timeout. Each step must be idempotent — inserting a dedup record before taking action.
- **Missing compensation for every step**: if step 3 fails and you only compensate step 2 but not step 1, you have an inconsistency. Map out the full compensation matrix before writing any code.
- **Not persisting saga state**: if the orchestrator crashes mid-saga and has no persistent state, it can't resume or compensate. Persist the current saga step and status durably.
- **Reading uncommitted saga state**: a saga in progress means transient inconsistency. External services should not read intermediate state values (e.g., stock should appear reserved, not deducted, until the saga completes).

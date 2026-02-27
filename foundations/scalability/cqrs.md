# CQRS (Command Query Responsibility Segregation)

## What it is

**CQRS** (Command Query Responsibility Segregation) is an architectural pattern that **separates the read model from the write model** of an application. Instead of a single model serving both reads (queries) and writes (commands), CQRS uses:

- **Commands**: mutate state — create, update, delete. Return nothing (or just an acknowledgment).
- **Queries**: read state — never mutate. Return data.

Each side can be independently optimized, scaled, and evolved.

The term was coined by Greg Young (2010), building on Bertrand Meyer's CQS (Command Query Separation) principle.

---

## How it works

### Traditional (CRUD) Model vs CQRS

**Traditional single model:**
```
Client → API → Same Model (ORM / DB) → Same Database
                    │
                  READ + WRITE at the same tables
```

**CQRS:**
```
               Command side (writes)
Client ──────► Command Handler → Write Model → Primary DB (normalized)
                                                     │
                                               Event / change propagation
                                                     │
               Query side (reads)                    ▼
Client ──────► Query Handler → Read Model ← Denormalized read store
              (SELECT-only)                (optimized projection)
```

### Write Side (Command Model)

Focused on business invariants and consistency:
- **Normalized schema** (prevent write anomalies, enforce constraints).
- **Transactional**: commands execute within ACID transactions.
- **Domain-driven**: commands represent business operations (`PlaceOrder`, `TransferFunds`, `PublishPost`), not CRUD.

```python
# Command
class PlaceOrderCommand:
    user_id: str
    items: List[OrderItem]
    shipping_address: Address

# Command handler — enforces business rules
class PlaceOrderHandler:
    def handle(self, cmd: PlaceOrderCommand):
        user = self.user_repo.get(cmd.user_id)
        inventory = self.inventory_service.check(cmd.items)

        if not inventory.all_available():
            raise InsufficientInventoryError()

        order = Order.create(cmd.user_id, cmd.items, cmd.shipping_address)
        self.order_repo.save(order)            # write to primary DB
        self.event_bus.publish(OrderPlaced(order))  # notify read side
```

### Read Side (Query Model)

Focused on query performance and flexibility:
- **Denormalized projections** purpose-built for the UI/API queries that will be made.
- **Multiple read models**: each query might have its own projection optimized for it.
- **Eventually consistent**: the read model is updated asynchronously after the write.

```python
# Read model for user's order history page
# Projection: pre-joined, pre-formatted, optimized for this page
class UserOrderSummaryProjection:
    order_id: str
    placed_at: datetime
    status: str
    total: Decimal
    item_count: int
    first_item_name: str

# read query is trivial — single table, pre-computed
def get_user_orders(user_id: str) -> List[UserOrderSummaryProjection]:
    return read_db.query(
        "SELECT * FROM user_order_summaries WHERE user_id = ? ORDER BY placed_at DESC",
        user_id
    )
```

### Synchronization: Keeping Read Models Updated

The write and read sides stay in sync via **events**:

```
Write side:
  Command → Write Model → Primary DB
                 └─► Event Bus (Kafka / domain events) ─────────┐
                                                                  │
Read side:                                                        ▼
  Event Consumer reads events → updates read model projections
  (ProjectionUpdater updates user_order_summaries table)
```

The read model is **eventually consistent** — there is a propagation delay (typically milliseconds) between a write and the read model being updated.

### Multiple Read Models Per Command

CQRS allows multiple independent read models:

```
Order Placed event
    ├── OrderHistoryProjection.handle()   → update user_order_summaries
    ├── SearchIndexProjection.handle()    → update Elasticsearch index  
    ├── RealtimeDashboard.handle()        → push to WebSocket dashboard
    └── AnalyticsProjection.handle()      → append to analytics stream
```

Each projection is optimized for its specific query. Adding a new view requires adding a new projection without touching write-side code.

### CQRS Without Event Sourcing

CQRS doesn't require Event Sourcing (though they're frequently combined). Simple CQRS:
- Write side: standard ACID transactions to primary DB.
- Synchronization: CDC (Change Data Capture) reads DB WAL → replicates to read model.
- Read side: separate read-optimized store (materialized view, Elasticsearch, Redis).

---

## Key Trade-offs

| Advantage | Disadvantage |
|---|---|
| Read and write sides can be scaled independently | Increased architectural complexity |
| Read models can be highly optimized for each query | Eventual consistency between write + read sides |
| Write side protects business invariants | More code: separate models, handlers, projections |
| New read projections without touching write code | Debugging is harder (event propagation chains) |
| Natural fit for event-driven architectures | Overkill for simple CRUD applications |

---

## When to use

CQRS is appropriate when:
- **Read and write loads are asymmetric**: e.g., millions of reads vs thousands of writes per day. Scale each side independently.
- **Read queries are complex and slow**: queries require expensive joins across many tables for display. Pre-compute a denormalized read model.
- **Multiple presentations of the same data**: a single write produces data needed in different shapes (API response, search index, dashboard, analytics). Event-driven projections are cleaner than coupling.
- **Domain is complex**: the write side benefits from rich domain models; the read side benefits from simple, query-optimized data.

**Don't use CQRS for:**
- Simple applications where a single consistent model per table suffices.
- Small teams where the complexity overhead isn't justified.
- Applications where read-after-write consistency is required for every operation (CQRS's eventual consistency violates this by default).

---

## Common Pitfalls

- **Forgetting the consistency lag**: after a command completes, the read model isn't immediately updated. If the UI reads immediately after a write, it may show stale data. Mitigate: optimistic UI updates, or read from the write model for a brief window post-command.
- **CQRS everywhere in a monolith**: CQRS adds complexity. Apply it only to the bounded contexts with complex query/write separation needs, not to the entire application.
- **Projection rebuild cost**: if the event log is lost or a new projection is needed, you must replay all past events. Event Sourcing (durable event store) makes projection rebuild feasible.
- **Tight coupling of command and query models**: if both sides share the same DB tables and just use different service layers, you've added process complexity without the actual benefits. True CQRS separates the physical data stores.

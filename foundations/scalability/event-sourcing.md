# Event Sourcing

## What it is

**Event Sourcing** is a data persistence pattern where the **state of a system is derived from a sequence of immutable events** rather than being stored directly as mutable state. Instead of `UPDATE accounts SET balance = 900`, you append `MoneyWithdrawn { amount: 100 }` to an event log. The current state is reconstructed by replaying all events from the beginning (or from a snapshot).

Event sourcing gives you a complete, immutable audit log of exactly how the system reached its current state.

---

## How it works

### Traditional State Storage vs Event Sourcing

**Traditional (mutable state):**
```sql
accounts: | id | balance | status   |
          | 1  | 900     | active   |
-- Current state only; history is gone
```

**Event Sourcing (immutable event log):**
```
account_events (append-only):
  AccountOpened    { account_id: 1, initial_balance: 1000 }  -- t=0
  MoneyDeposited   { account_id: 1, amount: 500 }            -- t=1
  MoneyWithdrawn   { account_id: 1, amount: 600 }            -- t=2

Current state = replay:
  balance = 1000 + 500 - 600 = 900 ✓
```

### Core Operations

#### Append an Event (Command Handling)

```python
class BankAccount:
    def withdraw(self, amount: Decimal):
        if self.balance < amount:
            raise InsufficientFundsError()
        # Emit event — do NOT mutate state directly
        event = MoneyWithdrawn(account_id=self.id, amount=amount, timestamp=now())
        self.apply(event)       # updates in-memory state
        self.pending_events.append(event)  # to be saved

    def apply(self, event):
        if isinstance(event, MoneyWithdrawn):
            self.balance -= event.amount
        elif isinstance(event, MoneyDeposited):
            self.balance += event.amount

# Persist the events
event_store.append(account.pending_events)
account.pending_events.clear()
```

#### Reconstruct State (Load an Aggregate)

```python
class BankAccountRepository:
    def load(self, account_id: str) -> BankAccount:
        events = event_store.load_events(account_id)       # all events for this ID
        account = BankAccount(account_id)
        for event in events:
            account.apply(event)                           # replay all events
        return account
```

#### Snapshots (Performance Optimization)

Replaying 1,000 events every time a record is loaded is slow. Snapshots periodically capture the current state to avoid full replay:

```python
def load(self, account_id: str) -> BankAccount:
    snapshot = snapshot_store.load_latest(account_id)
    if snapshot:
        account = BankAccount.from_snapshot(snapshot)
        # Only load events AFTER the snapshot
        events = event_store.load_events(account_id, after_version=snapshot.version)
    else:
        account = BankAccount(account_id)
        events = event_store.load_events(account_id)

    for event in events:
        account.apply(event)
    return account
```

Snapshot every 50–100 events; keep recent snapshots and the full event log.

### Event Store

The event store is an **append-only log** — events are never updated or deleted:

```
Minimum event store interface:
  append(stream_id: str, events: List[Event], expected_version: int)
      ← optimistic concurrency: fails if current version != expected
  load_events(stream_id: str, from_version: int = 0) -> List[Event]
  subscribe(stream_id: str) -> EventStream  ← pub/sub for projections
```

Implementations:
- **EventStoreDB**: purpose-built event store with subscription support.
- **PostgreSQL**: append-only events table with stream_id + version columns.
- **Kafka**: topics as event streams; offsets as versions.
- **DynamoDB**: partition key = stream_id; sort key = version.

```sql
-- PostgreSQL event store table
CREATE TABLE events (
  stream_id  TEXT NOT NULL,
  version    BIGINT NOT NULL,
  event_type TEXT NOT NULL,
  data       JSONB NOT NULL,
  metadata   JSONB,
  occurred_at TIMESTAMPTZ DEFAULT now(),
  PRIMARY KEY (stream_id, version)
);
```

### Projections (Read Models from Events)

Events are the source of truth; projections are derived read models:

```python
class AccountBalanceProjection:
    """Maintains a 'current_balances' table as a read model"""

    def on_account_opened(self, event: AccountOpened):
        db.execute("INSERT INTO balances (id, balance) VALUES (?, ?)",
                   event.account_id, event.initial_balance)

    def on_money_deposited(self, event: MoneyDeposited):
        db.execute("UPDATE balances SET balance = balance + ? WHERE id = ?",
                   event.amount, event.account_id)

    def on_money_withdrawn(self, event: MoneyWithdrawn):
        db.execute("UPDATE balances SET balance = balance - ? WHERE id = ?",
                   event.amount, event.account_id)
```

**Projection rebuild**: because events are immutable and permanent, any projection can be dropped and rebuilt by replaying the event log. New projections are built by replaying history.

### Temporal Queries — "What Was The Balance on Jan 15?"

Because all events are stored with timestamps, the state of the system at any past point can be reconstructed:

```python
def get_balance_at(account_id: str, as_of: datetime) -> Decimal:
    events = event_store.load_events(account_id, before=as_of)
    account = BankAccount(account_id)
    for event in events:
        account.apply(event)
    return account.balance
```

This is impossible in traditional mutable-state systems without an additional audit log.

---

## Event Sourcing with CQRS

Event sourcing and CQRS are frequently combined:
- **Write side (commands)**: commands produce domain events appended to the event store.
- **Projections (event handlers)**: consume events asynchronously and update read models.
- **Read side (queries)**: serve queries from the optimized read models.

```
Command → Aggregate → Event(s) appended to Event Store
                              │
                    ┌─────────┼────────────┐
                    ▼         ▼            ▼
              Projection-1  Projection-2  ...
              (SQL table)   (Elasticsearch)
```

---

## Key Trade-offs

| Advantage | Disadvantage |
|---|---|
| Full audit log — complete history of state changes | Higher complexity than traditional CRUD |
| New projections built from historical data | Eventual consistency between event store and projections |
| Temporal queries — "state as of" time | Event schema evolution is hard (old events must still be processable) |
| Natural fit with domain events + DDD | Event replay (loading aggregates) can be slow without snapshots |
| Debug by replaying events | Not suitable for all domains (simple CRUD often doesn't need it) |
| Immutable audit trail (compliance, debugging) | Tooling and infrastructure less mature than RDBMS |

---

## When to use

Event sourcing is well-suited for:
- **Domains requiring a full audit trail**: financial transactions, healthcare records, legal systems.
- **Complex business domains** where the history of state transitions carries semantic meaning.
- **Systems that need temporal queries**: "what was the state at time X?".
- **Event-driven microservices**: events serve as both the communication mechanism and the persistence model.
- **Compliance-heavy systems**: immutable event log satisfies regulatory audit requirements.

**Avoid event sourcing when:**
- Your domain is simple CRUD with no meaningful history semantics.
- Your team is unfamiliar with the pattern — the learning curve is steep.
- You need strong read-after-write consistency immediately after writes.

---

## Common Pitfalls

- **Event schema versioning**: events are permanent. When the event schema must change, you need an upcasting strategy — old events are mapped to the new schema as they're loaded.
- **Too many events per aggregate**: an aggregate with millions of events is slow to replay even with snapshots. Either use snapshots aggressively or reconsider the aggregate boundary.
- **No projection rebuild strategy**: when introducing a new projection or fixing a buggy one, you must replay all events. If the event store doesn't support efficient replay, this is painful. Build for it from the start.
- **Using event sourcing for everything**: event sourcing is a powerful pattern but overkill for most entities. Apply it only to aggregates where history and auditability matter.
- **Confused with event-driven architecture**: event sourcing is a persistence pattern. Event-driven architecture (using events for service communication) is a separate concern. They can be combined, but they're not the same thing.

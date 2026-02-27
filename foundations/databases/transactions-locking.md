# Transactions and Locking

## What it is

A **database transaction** is a unit of work that groups one or more read/write operations into an **atomic** sequence — either all operations succeed and are committed, or all are rolled back on failure. Transactions enforce **ACID** properties (Atomicity, Consistency, Isolation, Durability).

**Locking** (and related concurrency control mechanisms like MVCC) are the implementation mechanisms that allow multiple transactions to execute concurrently while still maintaining correct isolation guarantees.

---

## How it works

### Transaction Lifecycle

```
BEGIN;
  -- Multiple operations that form one logical unit
  UPDATE accounts SET balance = balance - 500 WHERE id = 1;
  UPDATE accounts SET balance = balance + 500 WHERE id = 2;
  INSERT INTO audit_log (action, amount) VALUES ('transfer', 500);
COMMIT;  -- All changes become visible atomically

-- On any failure:
ROLLBACK;  -- All changes are discarded
```

### ACID Recap

| Property | Guarantee |
|---|---|
| **Atomicity** | All-or-nothing; partial state is never visible or persisted |
| **Consistency** | Transaction leaves the database in a valid state (constraints satisfied) |
| **Isolation** | Concurrent transactions don't interfere with each other |
| **Durability** | Committed transactions survive crashes (persisted to disk via WAL) |

For depth, see [ACID Properties](../core-concepts/acid.md).

### Concurrency Problems Without Isolation

When transactions run concurrently without proper isolation, several anomalies can occur:

**1. Dirty Read**: Transaction A reads uncommitted changes from Transaction B. If B rolls back, A has read data that never existed.

```
T1: UPDATE users SET status = 'vip' WHERE id = 1;  <-- not committed yet
T2: SELECT status FROM users WHERE id = 1;          <-- reads 'vip' (dirty)
T1: ROLLBACK;                                        <-- status was never 'vip'
```

**2. Non-Repeatable Read**: Transaction A reads a row, Transaction B updates and commits it, then Transaction A reads the same row again — getting different values in a single transaction.

```
T1: SELECT balance FROM accounts WHERE id = 1;  --> $1000
T2: UPDATE accounts SET balance = 500 WHERE id = 1; COMMIT;
T1: SELECT balance FROM accounts WHERE id = 1;  --> $500 (changed!)
```

**3. Phantom Read**: Transaction A queries for rows matching a condition. Transaction B inserts new rows matching that condition and commits. Transaction A re-queries — new "phantom" rows appear.

```
T1: SELECT COUNT(*) FROM orders WHERE status = 'pending';  --> 5
T2: INSERT INTO orders (status) VALUES ('pending'); COMMIT;
T1: SELECT COUNT(*) FROM orders WHERE status = 'pending';  --> 6 (phantom!)
```

**4. Lost Update**: Two transactions read the same value, both modify it, and the second writer's change overwrites the first — the first write is lost.

```
T1: SELECT stock FROM products WHERE id = 1;  --> 10
T2: SELECT stock FROM products WHERE id = 1;  --> 10
T1: UPDATE products SET stock = 9 WHERE id = 1; (10 - 1)
T2: UPDATE products SET stock = 9 WHERE id = 1; (10 - 1) -- T1's decrement lost!
```

### Isolation Levels

SQL standard defines four isolation levels, each preventing a different set of anomalies:

| Isolation Level | Dirty Read | Non-Repeatable Read | Phantom Read |
|---|---|---|---|
| **Read Uncommitted** | Possible | Possible | Possible |
| **Read Committed** | Prevented | Possible | Possible |
| **Repeatable Read** | Prevented | Prevented | Possible |
| **Serializable** | Prevented | Prevented | Prevented |

**Read Committed** is the default in PostgreSQL and most RDBMS. It prevents dirty reads. Most applications run correctly at this level.

**Serializable** is the strongest. Transactions behave as if executed one at a time in serial order. Significant performance cost — avoid unless truly necessary.

### Optimistic Locking

Assumes conflicts are rare. Rows are not locked on read. At write time, the transaction checks whether the data has changed since it was read:

```sql
-- Add version column to the table
ALTER TABLE products ADD COLUMN version INT DEFAULT 0;

-- Read:
SELECT id, stock, version FROM products WHERE id = 1;
-- Returns: id=1, stock=10, version=5

-- Update with version check:
UPDATE products
SET stock = stock - 1, version = version + 1
WHERE id = 1 AND version = 5;
-- Affected rows: 1 → success
-- Affected rows: 0 → another transaction updated first → retry
```

- **Pros**: No blocking; higher concurrency; good when contention is low.
- **Cons**: Requires retry logic in the application; high-contention scenarios lead to many retries.

Use case: product inventory updates, user profile edits, CMS draft management.

### Pessimistic Locking

Assumes conflicts are likely. Rows are explicitly locked on read, preventing any concurrent modification:

```sql
-- Lock rows immediately on read
BEGIN;
SELECT * FROM accounts WHERE id IN (1, 2) FOR UPDATE;
-- Other transactions trying to UPDATE or SELECT FOR UPDATE these rows
-- will BLOCK until this transaction commits or rolls back
UPDATE accounts SET balance = balance - 500 WHERE id = 1;
UPDATE accounts SET balance = balance + 500 WHERE id = 2;
COMMIT;
```

- `SELECT FOR UPDATE`: exclusive lock (write intent).
- `SELECT FOR SHARE`: shared lock (read intent, allows other readers, blocks writers).

- **Pros**: Guarantees that only one transaction modifies a row; no retry logic needed.
- **Cons**: Reduces concurrency; risk of deadlocks; slow if transactions hold locks for a long time.

Use case: bank transfers, order fulfillment, any operation where "read-modify-write" correctness is critical and contention is high.

### MVCC (Multi-Version Concurrency Control)

Most modern RDBMS (PostgreSQL, MySQL/InnoDB) use MVCC to allow readers and writers to operate **without blocking each other**:
- On every write, a **new version** of the row is created rather than overwriting the old row.
- Readers see a **consistent snapshot** of the database as of the start of their transaction.
- Writers create new row versions; old versions are visible to active transactions that started before the write.

```
Transaction T1 starts: sees snapshot at T=100
   T2 writes and commits new row version at T=101
   T1 still sees the T=100 snapshot (old row version)
   T1 commits; old row version eligible for garbage collection
```

MVCC means a long-running `SELECT` (like a report) doesn't block writers, and vice versa. The cost is storage for old row versions (cleaned up by `autovacuum` in PostgreSQL).

### Deadlocks

A **deadlock** occurs when two transactions are each waiting for a lock held by the other:

```
T1 holds lock on Account A, waiting for lock on Account B
T2 holds lock on Account B, waiting for lock on Account A
→ Deadlock: neither can proceed
```

Database engines detect deadlocks automatically and kill one of the involved transactions (the "victim") with an error. The application must handle this by retrying.

**Prevent deadlocks by consistently acquiring locks in the same order:**
```sql
-- Always lock lower ID first:
SELECT * FROM accounts WHERE id = MIN(id1, id2) FOR UPDATE;
SELECT * FROM accounts WHERE id = MAX(id1, id2) FOR UPDATE;
```

---

## Key Trade-offs

| Choice | Gain | Cost |
|---|---|---|
| Higher isolation level | Fewer anomalies | Lower throughput, more blocking |
| Pessimistic locking | Guaranteed correctness | Reduced concurrency, deadlock risk |
| Optimistic locking | Higher concurrency | Retry logic required; poor under high contention |
| MVCC | Readers don't block writers | Old row versions require vacuuming/cleanup |
| Long-running transactions | Can span complex operations | Hold locks longer, increasing blocking |

---

## When to use

- **Pessimistic locking**: financial operations, inventory reservation, anywhere two concurrent operations on the same row are likely and data integrity is paramount.
- **Optimistic locking**: profile updates, CMS drafts — operations where simultaneous updates to the same row are rare and retry cost is low.
- **Serializable isolation**: financial reporting, complex invariant checks — use only when you've proven lower isolation levels cause correctness bugs.
- **Read Committed (default)**: correct for the vast majority of application use cases.

---

## Common Pitfalls

- **Long-held transactions**: opening a transaction, doing application work (API calls, user input), then committing holds database locks during the entire wait. Keep transactions short and database-only.
- **Forgetting to handle deadlock errors**: database clients throw specific errors on deadlock. Application code must catch and retry these.
- **SELECT inside transaction without FOR UPDATE**: opening a transaction, reading a value, then updating it based on what you read is a read-modify-write pattern that's vulnerable to lost updates unless you use `FOR UPDATE` or optimistic locking.
- **Using serializable everywhere "just to be safe"**: serializable reduces throughput significantly. Profile the actual anomalies you need to prevent and use the minimum isolation level that prevents them.
- **Ignoring MVCC bloat**: in PostgreSQL, high-churn tables with many MVCC versions and insufficient autovacuum activity lead to table bloat and performance degradation.

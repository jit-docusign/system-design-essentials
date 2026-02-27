# ACID Properties

## What it is

**ACID** is a set of four properties that guarantee database transactions are processed reliably, even in the presence of errors, crashes, or concurrent access. ACID is the foundation of relational databases and the reason they are trusted for financial systems, inventory management, and any domain where data correctness is non-negotiable.

- **A — Atomicity**: A transaction is all-or-nothing. Either all operations within it succeed, or none of them are applied.
- **C — Consistency**: A transaction brings the database from one valid state to another valid state. All defined rules, constraints, and invariants must hold before and after.
- **I — Isolation**: Concurrent transactions execute as if they were serial — each transaction is invisible to others until it commits.
- **D — Durability**: Once a transaction commits, its changes are permanent — even if the system crashes immediately after.

---

## How it works

### Atomicity

Databases implement atomicity using a **write-ahead log (WAL)** or **undo log**. Before any change is written to data pages, the intent and the old values are written to a log. If the transaction fails or the system crashes mid-transaction, the database uses the log to roll back all partial changes.

```
BEGIN TRANSACTION
  UPDATE accounts SET balance = balance - 500 WHERE id = 'A';  ← if this succeeds ...
  UPDATE accounts SET balance = balance + 500 WHERE id = 'B';  ← ... but this fails
ROLLBACK  ← atomicity ensures Account A is NOT debited
```

### Consistency

Consistency enforces database-level rules: NOT NULL constraints, UNIQUE constraints, foreign key relationships, check constraints, and application-defined invariants. The database engine validates these rules at commit time and rejects transactions that would violate them.

Note: ACID consistency is different from the "C" in CAP Theorem. CAP consistency is about distributed replication; ACID consistency is about satisfying database-defined invariants.

### Isolation

Isolation is the most complex of the four properties. It is controlled via **transaction isolation levels**, which trade off between correctness and concurrency:

| Isolation Level | Dirty Reads | Non-repeatable Reads | Phantom Reads |
|---|---|---|---|
| **Read Uncommitted** | Possible | Possible | Possible |
| **Read Committed** | Prevented | Possible | Possible |
| **Repeatable Read** | Prevented | Prevented | Possible |
| **Serializable** | Prevented | Prevented | Prevented |

- **Dirty Read**: Reading uncommitted changes from another transaction.
- **Non-repeatable Read**: Re-reading the same row within a transaction and getting a different value.
- **Phantom Read**: Re-running the same query and getting different rows because another transaction inserted/deleted data.

**Serializable** is the strongest level — it behaves as if all transactions ran one at a time. It is also the most expensive.

Most databases implement isolation using **Multi-Version Concurrency Control (MVCC)**: instead of locking rows, the database maintains multiple versions of a row. Readers see a consistent snapshot from transaction start time without blocking writers.

### Durability

Durability is guaranteed by writing committed data to non-volatile storage before acknowledging the commit. The WAL (Write-Ahead Log) is flushed to disk on commit. If the system crashes:
1. On restart, the database replays the WAL to recover committed transactions.
2. Partial (uncommitted) transactions are rolled back.

In cloud databases, durability is often achieved by **synchronous replication** to multiple availability zones before acknowledging the write.

---

## Key Trade-offs

| Property | Cost |
|---|---|
| **Atomicity** | Logging overhead; rollback complexity |
| **Consistency** | Constraint checks add validation latency |
| **Isolation (Serializable)** | Significant concurrency reduction; high lock contention |
| **Durability** | fsync to disk before ACK adds write latency (~1–10 ms per sync) |

The tension between ACID and performance is why **distributed NoSQL databases** adopted the BASE model — they relax or defer some ACID guarantees to achieve horizontal scalability and low latency.

---

## When to use

Choose a database with strong ACID guarantees when:
- Data correctness is non-negotiable: financial systems, payments, order management, healthcare records.
- Your application has complex multi-step operations that must succeed or fail together.
- You need foreign key constraints and referential integrity enforced by the database.
- Concurrent writes can corrupt data if not properly isolated (e.g., double-spending, overselling inventory).

---

## Common Pitfalls

- **Assuming "ACID" means everything is safe**: ACID protects against crashes and concurrency issues *at the database level*. Application-level bugs (wrong queries, missing transactions) are your responsibility.
- **Using serializable isolation by default**: It provides the strongest guarantees but can create deadlocks and dramatically reduce throughput. Use the **weakest isolation level** that is still correct for your use case.
- **Ignoring write skew**: Many teams assume Repeatable Read is sufficient. It is not when two transactions read overlapping data and write to different rows based on that read. Audit your critical paths for write skew, especially in scheduling, booking, and resource-allocation workflows.
- **Ignoring distributed ACID complexity**: ACID in a single-node database is well understood. Distributed ACID (across multiple databases or services) requires protocols like 2-Phase Commit or the Saga pattern — which have high coordination costs and failure modes of their own.
- **Conflating ACID consistency with CAP consistency**: They are completely different. ACID consistency is about constraints and invariants; CAP consistency is about distributed replicas agreeing on the same value.
- **Long-running transactions**: Holding transactions open for seconds or minutes causes lock contention, blocks other transactions, and can exhaust connection pools. Keep transactions short and targeted.
- **Assuming SSI is free**: SSI avoids most locking overhead but does abort transactions under detected conflicts. Applications must implement retry logic. Under high-conflict workloads, abort rates can be high enough that 2PL or application-level serialisation is preferable.

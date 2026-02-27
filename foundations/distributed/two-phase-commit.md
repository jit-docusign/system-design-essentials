# Two-Phase Commit (2PC)

## What it is

**Two-Phase Commit (2PC)** is a distributed algorithm that ensures multiple nodes in a distributed transaction either **all commit** or **all abort** — providing atomic commitment across multiple participants. It is the classic protocol for distributed ACID transactions.

Despite being well-understood and widely implemented (XA standard, MSDTC, distributed databases), 2PC has significant drawbacks — particularly its blocking behavior during coordinator failure — which is why alternatives like the Saga pattern are often preferred in modern microservices.

---

## How it works

### The Two Phases

**Phase 1: Prepare (Voting)**

The **Coordinator** sends a `PREPARE` message to all participants. Each participant:
- Checks whether it can commit (validates constraints, acquires locks).
- Writes all changes to a write-ahead log (WAL) — durable but not yet applied.
- Responds with `YES` (can commit) or `NO` (cannot commit).

```
Coordinator ──► Participant A: "Prepare to commit transaction T"
Coordinator ──► Participant B: "Prepare to commit transaction T"
Coordinator ──► Participant C: "Prepare to commit transaction T"

Participant A ──► Coordinator: YES (locks acquired, changes logged)
Participant B ──► Coordinator: YES
Participant C ──► Coordinator: NO (constraint violation)
```

**Phase 2: Commit/Abort**

If ALL participants voted YES: Coordinator sends `COMMIT`.
If ANY participant voted NO (or timeout): Coordinator sends `ABORT`.

```
Case 1: All YES → Coordinator sends COMMIT
  Participant A: applies changes, releases locks, ACK
  Participant B: applies changes, releases locks, ACK
  Participant C: applies changes (was YES), releases locks, ACK

Case 2: Any NO → Coordinator sends ABORT
  Participant A: rolls back logged changes, releases locks, ACK
  Participant B: rolls back, releases locks, ACK
  Participant C: already rejected → cleans up, ACK
```

### State Machine

```
Coordinator:        INIT → WAIT (sent PREPARE) → COMMIT or ABORT → DONE

Participant:        INIT → PREPARED (replied YES) → COMMITTED or ABORTED
                         → ABORTED   (replied NO)
```

### The Blocking Problem

2PC has a critical weakness: **if the coordinator fails after Phase 1 (after sending PREPARE) but before Phase 2 (before sending COMMIT/ABORT)**, participants are stuck in the PREPARED state — they hold locks and cannot proceed.

```
Coordinator sends PREPARE → all participants vote YES
Coordinator writes COMMIT to its log
Coordinator crashes BEFORE sending COMMIT messages

Participants:
  → Hold locks on resources
  → Don't know if they should commit or abort
  → Cannot unilaterally decide (violates atomicity)
  → Must wait for coordinator to recover ← BLOCKING
```

This is called the **blocking problem** of 2PC. While the coordinator is down, locked resources are unavailable, degrading system availability.

### 3PC (Three-Phase Commit)

3PC adds a `PREPARE-TO-COMMIT` phase between Prepare and Commit. This allows participants to abort unilaterally if the coordinator fails (they can detect the pre-commit phase was reached). However:
- 3PC is still vulnerable to network partitions.
- It's more complex.
- In practice, it's rarely used — Paxos/Raft-based commit is preferred for strong consistency with fault tolerance.

### XA Standard

XA is the X/Open standard interface for 2PC across heterogeneous resource managers (databases, message brokers). Supported by most enterprise databases.

```java
// Java XA distributed transaction (spanning two databases)
UserTransaction ut = (UserTransaction) ctx.lookup("java:comp/UserTransaction");

ut.begin();
try {
    // Both operations in the same 2PC transaction:
    inventoryDS.prepareStatement("UPDATE inventory SET qty=qty-1 WHERE id=?")
               .executeUpdate(productId);
    
    ordersDS.prepareStatement("INSERT INTO orders (product_id, user_id) VALUES (?,?)")
            .executeUpdate(productId, userId);
    
    ut.commit();   // 2PC coordinates commit across both databases
} catch (Exception e) {
    ut.rollback(); // 2PC coordinates rollback across both
}
```

### 2PC vs Saga

| Dimension | 2PC | Saga |
|---|---|---|
| Consistency model | Strong (ACID) | Eventual |
| Blocking | Yes (coordinator failure blocks participants) | No (compensating transactions) |
| Lock duration | Long-held during prepare | No long-lived locks |
| Coupling | Tight (coordinator + all participants) | Loose |
| Failure handling | Coordinator recovery (complex) | Compensating transactions |
| Scalability | Poor (locks + blocking) | Better |
| Use case | Centralized databases, legacy systems | Microservices with independent databases |

---

## Key Trade-offs

| Advantage | Disadvantage |
|---|---|
| Guarantees atomicity across multiple participants | Coordinator is a single point of failure and a bottleneck |
| All-or-nothing semantics: simple to reason about | Blocking: participants hold locks during coordinator failure |
| Well-understood protocol, widely supported (XA) | High latency: two network round-trips minimum |
| Strong consistency without application-level logic | Poor performance under high concurrency and failures |

---

## When to use

2PC is appropriate for:
- **Tightly-coupled systems with shared infrastructure**: multiple tables in different databases under the same team's control where strong consistency is required.
- **Legacy enterprise systems**: EJB/JTA/XA transactions in enterprise Java applications.
- **Database-internal distributed transactions**: many distributed databases (CockroachDB, TiDB, Google Spanner) use 2PC internally — but with Paxos/Raft to make the coordinator highly available and non-blocking.

Prefer **Saga** for:
- Microservices with independent databases.
- Systems where high availability is more important than strong consistency.
- Long-running business transactions.

---

## Common Pitfalls

- **Using 2PC across microservices without HA coordinator**: the coordinator is a SPOF unless backed by Paxos/Raft. A crashed coordinator blocks all participants indefinitely.
- **Long-held locks**: the prepare phase locks resources until Phase 2 completes. In a slow system, this causes contention. Design transactions to be as short as possible.
- **Not implementing coordinator recovery**: the coordinator must persist its log (COMMIT or ABORT decision) to disk before sending Phase 2 messages. On recovery, it re-sends the Phase 2 message to any non-acknowledged participants.
- **Assuming 2PC suits microservices**: 2PC requires a coordinator that can reach all participants. In microservices, each service has its own database, and a cross-service 2PC coordinator is impractical. Use the Saga pattern instead.

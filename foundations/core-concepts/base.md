# BASE Properties

## What it is

**BASE** is the set of properties adopted by many distributed NoSQL databases as an alternative to ACID. It trades strict correctness guarantees for higher availability, horizontal scalability, and lower latency.

- **B**asically **A**vailable: The system guarantees availability — it will always respond, but the response might reflect a stale or incomplete state.
- **S**oft State: The state of the system may change over time, even without new input, as nodes propagate and reconcile updates.
- **E**ventually Consistent: If no new updates are made, all replicas will converge to the same value — but there is no guarantee of when.

BASE was coined by Eric Brewer (who also proposed CAP) as a deliberate contrast to ACID, reflecting the philosophy of systems like Amazon Dynamo, Apache Cassandra, and CouchDB.

---

## How it works

### Basically Available

A BASE system is designed to continue serving requests even in degraded conditions:
- During a network partition, nodes respond with the data they have — even if it's potentially stale.
- During a partial failure, the available nodes keep serving reads and writes.
- The system chooses uptime and responsiveness over consistency guarantees.

This is the "AP" side of CAP: when forced to choose between availability and consistency, the system chooses availability.

### Soft State

In a strongly consistent system, the system's state is always deterministic and up-to-date on any read. In a BASE system, the state is "soft" — it can differ across replicas at any given instant, and the process of reconciling those differences is ongoing asynchronously.

Soft state means:
- Two clients reading from two different replicas may see different values simultaneously.
- Data written to one replica is propagating to others in the background.
- State may change even without a new user action (due to background synchronization).

### Eventually Consistent

The guarantee is: **if you stop writing, all replicas will eventually agree.** The time it takes to converge — the **replication lag** — depends on network conditions, system load, and the replication strategy.

```
Write "X = 2" to Node A at t=0

t=0:   Node A: X=2   |  Node B: X=1 (old)   |  Node C: X=1 (old)
t=50ms: Node A: X=2  |  Node B: X=2 (synced)|  Node C: X=1 (old)
t=100ms: Node A: X=2 |  Node B: X=2          |  Node C: X=2 (synced)
```

Between t=0 and t=100ms, reads from different nodes may return different values. After t=100ms, all nodes agree — they have "eventually converged."

### Conflict Resolution

When two nodes accept conflicting writes (both receive different values for the same key during a partition), they must reconcile after the partition heals. Common strategies:

| Strategy | Description |
|---|---|
| **Last Write Wins (LWW)** | The write with the most recent timestamp wins; other writes are discarded |
| **Multi-value / siblings** | Keep all conflicting versions and return them to the client for resolution |
| **Application-level merge** | Application defines how to merge conflicting states (e.g., union of sets) |
| **CRDTs** | Data structures designed such that any order of operations converges to the same result |

---

## BASE vs ACID

| Property | ACID | BASE |
|---|---|---|
| **Availability** | May reject requests to preserve consistency | Always responds |
| **Consistency** | Immediate and strict | Eventually convergent |
| **Concurrency model** | Transactions with isolation levels | Optimistic, conflict resolution post-write |
| **Failure behavior** | Rejects or rolls back | Accepts, reconciles later |
| **Horizontal scale** | Difficult (requires distributed transactions) | Designed for it |
| **Typical use case** | Financial, transactional, relational | Social, analytics, high-throughput, global |

---

## When to use

Choose BASE / eventually consistent systems when:
- **Availability is more important than immediate correctness**: user activity feeds, view counts, like counts, shopping cart drafts.
- **Data can be reconciled or merged**: inventory counts that can be corrected after the fact, document edits via CRDT.
- **Write throughput is a primary concern**: logging, analytics, time-series metrics, IoT event streams.
- **Global distribution is required**: eventual consistency avoids the round-trip coordination costs of strong consistency across continents.

Avoid BASE when:
- Conflicts cannot be tolerated or resolved: account balances, ticket inventory, medical records.
- The application cannot handle or reason about stale reads.

---

## Common Pitfalls

- **Treating "eventually consistent" as a magic escape hatch**: Eventual consistency without a well-defined conflict resolution strategy can lead to permanent data corruption or data loss.
- **Assuming eventual consistency is always fast**: "Eventually" could be milliseconds or could be minutes/hours if a partition lasts a long time or nodes are under load.
- **Not informing clients about staleness**: If a user writes data and immediately reads it back from a different replica, they may not see their own write — this violates "read-your-writes" and feels like a bug. Design UX and application logic to account for this.
- **Underestimating the complexity of conflict resolution**: Last Write Wins is simple but lossy. More correct strategies (siblings, CRDTs) are complex to implement correctly.
- **Using BASE for everything**: Some parts of a system truly require ACID (the payments service). Use BASE selectively where the trade-offs are acceptable, rather than as a universal default.

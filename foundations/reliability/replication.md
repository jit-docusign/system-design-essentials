# Replication

## What it is

**Replication** is the process of maintaining multiple copies of the same data on different nodes, servers, or geographic locations. It serves two primary purposes:
1. **Fault tolerance / High availability**: if one replica fails, others can continue serving requests.
2. **Read scalability**: distribute read traffic across multiple replicas to reduce load on any single node.

Replication is one of the most fundamental primitives in distributed systems design — almost every scalable system relies on it in some form.

---

## How it works

### Single-Leader Replication (Primary-Replica / Master-Slave)

One node is designated the **leader/primary** — all writes go here. The leader propagates changes to **followers/replicas**, which serve read traffic.

```
Writes:
  Client → Leader (writes accepted)
              ↓ replication
         Replica 1   Replica 2   Replica 3

Reads:
  Client → Replica 1 or 2 or 3 (reads distributed)
```

**Replication log**: The leader writes changes to a replication log (WAL or statement log). Followers tail this log and apply the same operations in order.

**Write path**: 1 node (the leader) — simple, no write conflicts.  
**Read path**: N nodes (replicas) — horizontally scalable reads.

### Synchronous vs Asynchronous Replication

**Synchronous replication:**
- The leader waits for a replica to acknowledge the write before confirming to the client.
- Guarantees that at least one replica is always up-to-date with the leader.
- Higher write latency; if the sync replica is slow, it slows down all writes.
- **No data loss** if the leader fails immediately after a commit.

**Asynchronous replication:**
- The leader acknowledges the write without waiting for replicas.
- Lower write latency; replicas catch up independently.
- **Replication lag**: replicas may be behind by milliseconds to seconds (or more under load).
- **Risk of data loss**: if the leader fails before replication, recent writes are lost.

**Semi-synchronous (hybrid):**
- At least one replica must acknowledge synchronously; others replicate asynchronously.
- Used in MySQL semi-synchronous replication. Balances durability with performance.

### Replication Lag

Even with asynchronous replication, lag is normally small (sub-second). But under load or during network issues, it can grow significantly. Effects:
- **Read-after-write inconsistency**: A user writes to the leader, then reads from a replica that hasn't caught up yet — they don't see their own write.
- **Monotonic read violations**: A user reads from Replica A, then Replica B — Replica B is behind Replica A, so data appears to go backward.
- **Causality violations**: A user reads a reply to a post but hasn't yet seen the original post.

### Multi-Leader Replication (Multi-Master)

Multiple nodes accept writes. Each leader replicates its writes to the others.

**Use cases:**
- Multi-datacenter deployments: each datacenter has its own leader for low write latency.
- Offline-capable applications (e.g., a calendar app that writes while offline then syncs).

**Problem: Write Conflicts**

If two leaders simultaneously accept writes to the same key, you get a conflict:

```
Leader A: X = "Alice" at t=1
Leader B: X = "Bob"   at t=1
Both replicate to each other — which wins?
```

Conflict resolution strategies:
- **Last Write Wins (LWW)**: Use timestamps; most recent write wins. Risk: clock skew can cause incorrect resolution; writes can be silently discarded.
- **Merge**: Combine values (e.g., union of two sets in a CRDT).
- **Application-level resolution**: Store all conflicting versions and let the application or user resolve.

### Leaderless Replication (Dynamo-style)

No single leader. Any node can accept reads and writes. A quorum of nodes must agree for an operation to succeed.

Given N replicas:
- **W** = write quorum (minimum nodes that must acknowledge a write)
- **R** = read quorum (minimum nodes that must be queried for a read)

For strong consistency: **W + R > N**

```
N=3, W=2, R=2: W + R = 4 > 3 → at least 1 node overlap → always see latest write
N=3, W=1, R=1: W + R = 2 = N → no guarantee of seeing latest write (AP behavior)
```

**Read repair**: when a read detects stale data on one of the queried replicas, it updates it in the background.

**Anti-entropy**: background process continuously compares replicas using Merkle trees and reconciles differences.

---

## Key Trade-offs

| Strategy | Write latency | Read scalability | Consistency | Complexity |
|---|---|---|---|---|
| Single-leader + sync | High | High | Strong | Low |
| Single-leader + async | Low | High | Eventual | Low |
| Multi-leader | Low | High | Eventual (conflict risk) | High |
| Leaderless (quorum) | Tunable | Tunable | Tunable | High |

---

## When to use

**Single-leader + async replicas**: Most common; suitable for read-heavy workloads where replication lag is acceptable (social media, content sites).

**Single-leader + sync replica**: When durability is non-negotiable and losing even one write is unacceptable (financial systems, inventory).

**Multi-leader**: Multi-datacenter setups where local writes (low latency) are more important than instant global consistency.

**Leaderless**: Systems designed for high availability and partition tolerance at massive scale (Cassandra, DynamoDB).

---

## Common Pitfalls

- **Replication lag causing read-after-write bugs**: Route reads of user's own data to the primary, or use a session token to route to a replica that has caught up.
- **Promoting the wrong replica**: In async replication, the replica most up-to-date isn't always the most recently promoted. The new leader may be missing the last few seconds of writes from the failed primary — these are lost. Accepted trade-off for asynchronous setups.
- **Not monitoring replication lag**: Lag can grow silently under load. Alert when it exceeds a threshold meaningful to your SLAs.
- **Multi-master without conflict resolution**: Concurrent conflicting writes without a defined resolution strategy permanently corrupt data. Design conflict resolution before enabling multi-master writes.
- **Confusing replication (availability) with backups (disaster recovery)**: A replica that is continuously replicating from a primary will also replicate a DELETE TABLE. Backups are not a substitute for each other.

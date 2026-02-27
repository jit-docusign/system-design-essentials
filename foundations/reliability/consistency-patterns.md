# Consistency Patterns

## What it is

**Consistency patterns** describe what guarantees a distributed system makes about when and how data written to one node becomes visible to readers on other nodes. Unlike ACID consistency (which is about database constraints), distributed consistency is about **how replicas converge** and **what a client can observe** during and after a write.

Understanding these patterns is essential for choosing the right database, designing cache invalidation strategies, and reasoning about concurrent writes.

---

## How it works

### The Consistency Spectrum

From strongest to weakest:

#### 1. Strong / Linearizable Consistency

Every read reflects the most recent write. Operations appear as if they execute instantaneously on a single copy of the data.

```
Client A writes X = 2 at t=1
Client B reads X at t=2 → always gets X = 2
```

- **How it's achieved**: Synchronous replication to quorum; all reads go through the primary or require quorum agreement.
- **Cost**: High latency (cross-node coordination); reduced availability during partitions.
- **Examples**: Zookeeper, etcd, Google Spanner (using TrueTime), single-node RDBMS.

#### 2. Sequential Consistency

All nodes see operations in the same order, but that order doesn't have to match real wall-clock time.

```
All replicas agree: A happened before B — but "A happened before B" might not reflect real time.
```

- Weaker than linearizability (no real-time guarantee), but all clients see the same ordering.
- Rare in practice as a named guarantee; mostly a theoretical stepping stone.

#### 3. Causal Consistency

Operations that are causally related (one causes another) are seen in the correct order by all nodes. Unrelated (concurrent) operations may be seen in any order.

```
Alice posts a message (write W1)
Bob reads Alice's message (read R1 — causally depends on W1)
Bob replies (write W2 — causally depends on W1 and R1)

All nodes will see W1 before W2.
Concurrent writes by unrelated users may be seen in any order.
```

- **How it's achieved**: Vector clocks or dependency tracking.
- **Use case**: Comment threads, collaborative editing, social media replies.

#### 4. Read-Your-Writes (Session Consistency)

A client is guaranteed to see the effects of its own writes in subsequent reads — even if other clients may not yet.

```
Client A writes X = 5
Client A reads X → always sees X = 5 (their own write)
Client B reads X → may still see X = 3 (hasn't replicated yet)
```

- **How it's achieved**: Route subsequent reads to the same replica the write went to (sticky sessions), or use a write token/version to route to replicas that have caught up.
- **Use case**: Profile updates, preferences — the user who made the change expects to see it immediately.

#### 5. Monotonic Reads

Once a client has read a value, it will never read an older value in a later read — even on a different replica.

```
Client reads X = 5 from Replica A
Client reads X from Replica B → will see X ≥ 5, never X = 3
```

- Prevents the disorienting "time travel" experience where data appears to go backward.
- **How it's achieved**: Sticky sessions to the same replica, or version-gating reads.

#### 6. Eventual Consistency

If no new writes occur, all replicas will converge to the same value — but there is no guarantee of when.

```
Write X = 5 to Node A at t=0
Node B may still return X = 3 at t=200ms
Node B will return X = 5 at t=500ms (after replication)
```

- **Cost**: Any write may be temporarily invisible to some readers.
- **Use case**: DNS, social media counts, shopping carts, analytics.
- Requires a conflict resolution strategy when concurrent conflicting writes occur.

### Visualizing the Spectrum

```
← Stronger guarantees, higher latency          Weaker guarantees, lower latency →

Linearizable → Sequential → Causal → Read-Your-Writes → Monotonic Reads → Eventual
```

---

## Key Trade-offs

| Pattern | Latency | Availability | Difficulty |
|---|---|---|---|
| Linearizable | High (quorum sync) | Lower | Simple for apps |
| Causal | Medium | Higher | Moderate |
| Read-your-writes | Low | High | Moderate (routing) |
| Monotonic reads | Low | High | Moderate (routing) |
| Eventual | Lowest | Highest | Complex (conflicts) |

---

## When to use each

| Pattern | Use case |
|---|---|
| **Linearizable** | Distributed locks, leader election, configuration, bank balances |
| **Causal** | Social feeds, comment threads, collaborative tools |
| **Read-your-writes** | Any user-facing write (profile updates, settings, posting content) |
| **Monotonic reads** | Paginated reads, dashboards — anything where backward time is confusing |
| **Eventual** | Like/view counts, DNS, caches, activity feeds, analytics |

---

## Common Pitfalls

- **Assuming eventual consistency is safe without conflict resolution**: Without a defined merge strategy, concurrent conflicting writes can cause permanent data corruption.
- **Not providing read-your-writes for user-facing writes**: Users who submit a form and then don't see their change immediately file "data loss" bugs. Always guarantee read-your-writes for user-initiated mutations.
- **Using linearizable consistency everywhere**: It's the safest; it's also the most expensive. Profile first; apply strong consistency only where the application logic requires it.
- **Confusing session consistency with global consistency**: Read-your-writes only guarantees a user sees their own writes — not the writes of all other users. Don't rely on it as a substitute for global consistency.

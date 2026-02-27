# Vector Clocks

## What it is

A **vector clock** is a data structure used in distributed systems to **track causality** — to determine whether one event happened before another, after another, or if they are concurrent (causally unrelated). Unlike physical clocks (wall clocks), vector clocks don't measure time; they measure the causal ordering of events.

Physical clocks cannot be trusted for causal ordering in distributed systems — different nodes have different clock rates and drift. Vector clocks solve this without synchronized clocks.

---

## How it works

### The Problem: Causal Ordering

```
Node A sends message M to Node B at 10:00:01.000 (A's clock)
Node B receives M at  09:59:59.999 (B's clock — drifted behind)

Physical time says: B received M before A sent it → impossible!

Vector clocks: track causality correctly regardless of clock drift.
```

### Vector Clock Structure

Each node maintains a vector (array) of counters, one per node:

```
In a 3-node system [A, B, C]:
  Node A's vector:  [A=2, B=1, C=0]
  Node B's vector:  [A=1, B=3, C=1]
  Node C's vector:  [A=0, B=0, C=1]
```

**Rules:**
1. When a node performs a local event: increment its own counter.
2. When a node sends a message: increment own counter, attach the full vector to the message.
3. When a node receives a message: update each component to `max(local[i], received[i])`, then increment own counter.

### Full Example

```
Initial state:   A=[0,0,0]   B=[0,0,0]   C=[0,0,0]
                 (indices:    A   B   C)

Step 1: A performs local event → A=[1,0,0]
Step 2: A sends message to B, attaches [1,0,0]
         B receives: max([0,0,0], [1,0,0]) = [1,0,0], then B increments → B=[1,1,0]
Step 3: B sends message to C, attaches [1,1,0]
         C receives: max([0,0,0], [1,1,0]) = [1,1,0], then C increments → C=[1,1,1]
Step 4: A sends message to C, attaches [1,0,0]  (A hasn't seen B's updates)
         C receives: max([1,1,1], [1,0,0]) = [1,1,1], then C increments → C=[1,1,2]

On Step 4: C's vector already has B=1 from Step 3, so A's message doesn't update it.
```

### Comparing Vector Clocks (Causality)

Given two vectors V1 and V2:

- **V1 happened before V2** (`V1 < V2`): V1[i] ≤ V2[i] for all i, AND V1[j] < V2[j] for at least one j.
- **V2 happened before V1** (`V2 < V1`): reverse of above.
- **Concurrent** (`V1 || V2`): neither V1 < V2 nor V2 < V1. Events happened independently without causal relationship.

```python
def compare_vectors(v1, v2):
    """Returns 'before', 'after', or 'concurrent'."""
    v1_leq = all(v1[i] <= v2[i] for i in range(len(v1)))
    v2_leq = all(v2[i] <= v1[i] for i in range(len(v1)))
    
    if v1_leq and not v2_leq:
        return 'before'   # v1 happened before v2
    elif v2_leq and not v1_leq:
        return 'after'    # v2 happened before v1
    elif v1_leq and v2_leq:
        return 'equal'    # same event
    else:
        return 'concurrent'  # causally unrelated

v1 = [1, 0, 0]
v2 = [1, 1, 0]
print(compare_vectors(v1, v2))  # 'before' (v1 < v2)

v3 = [2, 0, 0]
v4 = [0, 2, 0]
print(compare_vectors(v3, v4))  # 'concurrent'
```

### Use Case: Conflict Detection in Distributed Key-Value Stores

Amazon Dynamo uses vector clocks to detect and resolve write conflicts when multiple replicas accept concurrent writes:

```
Client writes key=user:123:profile with value={"name": "Alice"}
  → Replica 1 accepts write: clock [R1=1, R2=0, R3=0]
  → Replica 2 accepts write: clock [R1=0, R2=1, R3=0]  (concurrent — network partition)

Client reads user:123:profile:
  Gets both versions:
    v1 = {"name": "Alice"} with clock [1,0,0]
    v2 = {"name": "Alice Smith"} with clock [0,1,0]
  
  compare_vectors([1,0,0], [0,1,0]) → concurrent

  Client must reconcile: choose one, merge, or ask user.
  Client reconciles → writes merged value with clock [1,1,1].
```

### Vector Clocks vs Lamport Timestamps

| | Lamport Timestamp | Vector Clock |
|---|---|---|
| Structure | Single integer | Array of N integers |
| Causality detection | Partial only: if A→B then L(A) < L(B), but L(A) < L(B) doesn't mean A→B | Full: detects both causality and concurrency |
| Size | O(1) | O(N) — grows with number of nodes |
| Can detect concurrency? | No | Yes |

For detecting concurrent writes and conflicts, vector clocks are necessary; Lamport timestamps only establish a partial order.

### Dotted Version Vectors (Riak)

Riak uses **Dotted Version Vectors** — an improvement over basic vector clocks that avoids "sibling explosion" (too many concurrent versions) by more precisely tracking which writes are causal ancestors.

---

## Key Trade-offs

| Advantage | Disadvantage |
|---|---|
| Correctly identifies causality without synchronized clocks | Vector size grows O(N) — expensive with many nodes |
| Detects concurrent events for conflict resolution | Requires node IDs embedded in every message |
| Foundational for eventual consistency systems | Clock size can bloat in large dynamic clusters |
| No central coordinator needed | Complex to reason about in debugging |

---

## When to use

Vector clocks are used when:
- You need to detect **causally concurrent writes** for conflict resolution in eventually consistent stores (Dynamo, Riak, Cassandra LWW alternatives).
- You need to establish **causal ordering** of events across nodes for reasoning about distributed traces or debugging.
- Building **CRDT-based systems** where merger of concurrent state requires knowing which updates are causal.

For simpler ordering needs (e.g., event log timestamping), **Lamport timestamps** or **hybrid logical clocks (HLC)** are lighter.

---

## Common Pitfalls

- **Using physical clocks for causality**: wall clocks can go backward (NTP adjustments), can drift, and are different on each machine. Physical time cannot establish causal order across nodes. Use vector clocks or logical clocks.
- **Unbounded vector growth**: in systems where node IDs are dynamic (containers autoscaling), vector size can grow indefinitely. Use versioned vector pruning or Dotted Version Vectors.
- **Confusing concurrent with conflicting**: concurrent means causally unrelated, not necessarily conflicting in value. Two reads of different keys that happen concurrently are fine. Conflict only matters when concurrent writes target the same key.

# CRDTs — Conflict-Free Replicated Data Types

## What it is

A **CRDT** (Conflict-Free Replicated Data Type) is a data structure that can be concurrently updated on multiple nodes without coordination, and **automatically merges all updates into a consistent final state** — without conflicts, no matter what order updates are applied.

CRDTs are the mathematical foundation for eventual consistency **without conflicts**: no locks, no consensus round-trips, no human-in-the-loop conflict resolution. If you design your data types using CRDTs, concurrent updates from any number of nodes will always converge to the same correct final state.

Used in: Riak (counters, sets), Redis (Redis CRDT with RedisGears), Cassandra (counters), collaborative editing tools (Google Docs, Figma), Automerge/Yjs (offline-first apps).

---

## How it works

### The Core Insight

For a CRDT to work without conflicts, all merge operations must satisfy three mathematical properties:

```
Commutative:  merge(A, B) = merge(B, A)        (order doesn't matter)
Associative:  merge(merge(A, B), C) = merge(A, merge(B, C))  (grouping doesn't matter)
Idempotent:   merge(A, A) = A                  (applying same update twice = once)
```

If merge satisfies these three, any order of receiving and merging updates produces the same result.

### Two Families of CRDTs

**CvRDT (Convergent CRDT / State-based)**: nodes exchange full state and merge. 
**CmRDT (Commutative CRDT / Operation-based)**: nodes exchange operations; operations must be commutative.

CvRDTs are simpler to reason about. CmRDTs are more bandwidth efficient (send small ops, not full state).

### Common CRDT Types

#### G-Counter (Grow-only Counter)

A distributed counter that only increments. Each node has its own count slot.

```python
class GCounter:
    def __init__(self, node_id, num_nodes):
        self.node_id = node_id
        self.counts = [0] * num_nodes  # one slot per node
    
    def increment(self):
        self.counts[self.node_id] += 1
    
    def value(self):
        return sum(self.counts)
    
    def merge(self, other):
        """Merge by taking the max of each slot."""
        self.counts = [max(a, b) for a, b in zip(self.counts, other.counts)]

# Concurrent increments on 3 nodes:
nodeA = GCounter(0, 3)
nodeB = GCounter(1, 3)
nodeC = GCounter(2, 3)

nodeA.increment()  # nodeA.counts = [1, 0, 0] → value = 1
nodeA.increment()  # nodeA.counts = [2, 0, 0] → value = 2
nodeB.increment()  # nodeB.counts = [0, 1, 0] → value = 1

# After merge:
nodeA.merge(nodeB)  # counts = [2, 1, 0] → value = 3

# Safe: merge is idempotent
nodeA.merge(nodeB)  # again → counts = [2, 1, 0] → value = 3  (no change)
```

#### PN-Counter (Increment and Decrement Counter)

Two G-Counters: one for increments (P), one for decrements (N). Value = P.value() - N.value().

#### G-Set (Grow-only Set)

A set that only allows adds. Never removes. Merge = union.

```python
class GSet:
    def __init__(self):
        self.elements = set()
    
    def add(self, element):
        self.elements.add(element)
    
    def contains(self, element):
        return element in self.elements
    
    def merge(self, other):
        self.elements = self.elements | other.elements  # union

nodeA_set = GSet()
nodeB_set = GSet()

nodeA_set.add("alice")
nodeB_set.add("bob")

nodeA_set.merge(nodeB_set)  # {alice, bob}
nodeB_set.merge(nodeA_set)  # {alice, bob}  ← same result, regardless of order
```

#### 2P-Set (Two-Phase Set: add + remove, no re-add)

Two G-Sets: add-set and remove-set. An element is in the set iff it's in add-set AND not in remove-set. Once removed, it cannot be re-added.

#### OR-Set (Observed-Remove Set — supports add + remove + re-add)

Each `add` operation creates a unique tag for the element. `remove` removes all observed tags. If an element is added concurrently with its removal, the add wins (by using a tag not in the remove set).

```
Node A: add("alice") → {("alice", uuid-1)}
Node B: removes "alice" → removes {("alice", uuid-1)}

Concurrent: Node A adds "alice" again → {("alice", uuid-2)}
After merge: "alice" is present (uuid-2 survived the remove of uuid-1)

This is how Redis CRDT sets and Riak Sets work.
```

#### LWW-Register (Last-Write-Wins Register)

Stores a single value; concurrent writes are resolved by choosing the write with the highest timestamp.

```python
class LWWRegister:
    def __init__(self):
        self.value = None
        self.timestamp = 0
    
    def write(self, value, timestamp):
        if timestamp > self.timestamp:
            self.value = value
            self.timestamp = timestamp
    
    def merge(self, other):
        if other.timestamp > self.timestamp:
            self.value = other.value
            self.timestamp = other.timestamp
```

LWW is simple but can lose writes if timestamps collide or if physical clocks are unreliable. Used extensively in Cassandra.

### Collaborative Text Editing (RGA / Logoot / Automerge)

Real-time collaborative editing (Google Docs model) uses sequence CRDTs:

```
Problem: If Node A inserts "B" at position 1 and Node B inserts "C" at position 1 simultaneously,
  merging by position causes interleaving bugs.

RGA (Replicated Growable Array) / Logoot:
  Each character gets a globally unique ID and a "tombstone" position.
  Characters are sorted by their IDs consistently on all nodes.
  Deletions mark characters as invisible (tombstone), not physically removed.
  
  Result: all nodes converge to the same document, concurrent inserts produce consistent ordering.
```

Libraries: **Automerge** (JavaScript/Rust), **Yjs** (JavaScript/Go) — implement sequence and map CRDTs for offline-first collaborative apps.

---

## Key Trade-offs

| Advantage | Disadvantage |
|---|---|
| Eventual consistency without conflicts | Not all data structures can be expressed as CRDTs |
| No coordination, locks, or consensus needed | Richer CRDTs (OR-Set, sequences) have complex implementations |
| Works under partition — nodes can update offline | State-based CRDTs have growing metadata overhead |
| Concurrent updates always converge | LWW discards some updates — not suitable where all writes matter |

---

## When to use

CRDTs are appropriate for:
- **Distributed counters**: page views, likes, inventory counts — use PN-Counter.
- **Shared sets/lists**: user presence sets, collaborative whiteboards — use OR-Set.
- **Offline-first applications**: mobile apps that sync on reconnect — Automerge, Yjs.
- **Multi-region replication**: accept writes anywhere, merge on sync — Riak, Redis Enterprise CRDT.
- **Real-time collaboration**: Google Docs-style concurrent editing — sequence CRDTs.

Not suitable for:
- **Unique constraints across nodes** (e.g., unique usernames) — requires coordination.
- **Financial ledgers** requiring exact audit trail — use event sourcing + consensus.
- **Complex transactional semantics** — use 2PC or Sagas.

---

## Common Pitfalls

- **Using LWW when all writes matter**: LWW silently discards the lower-timestamped concurrent write. If both writes have business meaning (two users both adding an item to cart simultaneously), use an OR-Set or merge strategy that preserves both.
- **Metadata growth in OR-Sets**: each add creates a new UUID tag. Without garbage collection (pruning observed tags), OR-Set metadata grows unboundedly.
- **Forgetting idempotency in CmRDTs**: operation-based CRDTs rely on at-least-once delivery and require operations to be idempotent (safe to replay). Ensure duplicate operations are filtered at the merge layer.

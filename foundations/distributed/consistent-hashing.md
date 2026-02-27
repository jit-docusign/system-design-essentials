# Consistent Hashing

## What it is

**Consistent hashing** is a technique for distributing data across a dynamic set of nodes such that when nodes are added or removed, **only a minimal fraction of keys need to be remapped** — rather than remapping nearly all keys as in naive modular hashing.

This is essential for distributed caches, distributed hash tables, and database sharding where nodes join and leave constantly, and remapping all keys would cause massive cache misses or data migration overhead.

---

## How it works

### The Problem with Naive Hashing

```
Naive modular hashing:  node = hash(key) % N

With N=3 nodes: key "user-123" → hash=456 → 456 % 3 = 0 → Node 0

Add a 4th node (N=4):
  key "user-123" → 456 % 4 = 0 → Node 0 (same, lucky)
  key "user-456" → 789 % 3 = 0 → Node 0, but 789 % 4 = 1 → Node 1 ← MOVED
  key "user-789" → 234 % 3 = 0 → Node 0, but 234 % 4 = 2 → Node 2 ← MOVED

Adding one node remaps ~75% of all keys.
In a distributed cache: 75% cache miss rate until data is repopulated. 
```

### The Hash Ring

Consistent hashing maps both **nodes** and **keys** onto a circular ring (hash space from 0 to 2³²-1):

```
         hash(key) → position on ring
                    
              0
           /     \
  2³²-1           key K1 (hash=10)
          \     /
         2³¹

Nodes are also placed by hash(node_id):
  Node A: hash = 100
  Node B: hash = 200
  Node C: hash = 300

Hash Ring:
  0 ──── 100 ─────── 200 ─────── 300 ──── 0 (wraps)
          A            B           C

Key assignment: each key is assigned to the first node clockwise from its position.
  Key K1 (hash=50):  → clockwise → hits A first → assigned to A
  Key K2 (hash=150): → clockwise → hits B first → assigned to B
  Key K3 (hash=250): → clockwise → hits C first → assigned to C
```

### Adding/Removing Nodes

**Adding a node D at hash=150:**
```
Before: Key K2 (hash=150) → B
After:  Key K2 (hash=150) → D (D is now at 150, between A(100) and B(200))

Only keys between A(100) and D(150) migrate from B to D.
All other keys are unaffected.
```

**Expected keys remapped when adding 1 node to N-node cluster: `1/N` of all keys.**

For N=100 nodes: adding one node remaps only 1% of keys (vs 99% with modular hashing).

### Virtual Nodes (Vnodes)

A single physical node on the ring creates an imbalanced distribution — by chance, some nodes handle more keys than others. **Virtual nodes** solve this by assigning each physical node multiple positions on the ring:

```
Each physical node → V virtual node positions (default: 150 in Cassandra)

Physical nodes: A, B, C
Virtual nodes on ring (V=3 per node):
  0 ─ A1 ─ B2 ─ C1 ─ A2 ─ B1 ─ C3 ─ A3 ─ B3 ─ C2 ─ 0

Key assignment: first virtual node clockwise → maps to its physical node

Benefits:
  - Uniform distribution across physical nodes
  - When a node is added, it takes small slices from EVERY existing node (not just neighbors)
  - Easy to assign different node weights (more powerful nodes get more vnodes)
```

```python
import hashlib
import bisect

class ConsistentHashRing:
    def __init__(self, nodes=None, virtual_nodes=150):
        self.virtual_nodes = virtual_nodes
        self.ring = {}         # hash → node_id
        self.sorted_keys = []  # sorted hash values for binary search
        
        for node in (nodes or []):
            self.add_node(node)
    
    def _hash(self, key: str) -> int:
        return int(hashlib.md5(key.encode()).hexdigest(), 16)
    
    def add_node(self, node_id: str):
        for i in range(self.virtual_nodes):
            vnode_key = f"{node_id}:vnode-{i}"
            h = self._hash(vnode_key)
            self.ring[h] = node_id
            bisect.insort(self.sorted_keys, h)
    
    def remove_node(self, node_id: str):
        for i in range(self.virtual_nodes):
            vnode_key = f"{node_id}:vnode-{i}"
            h = self._hash(vnode_key)
            del self.ring[h]
            self.sorted_keys.remove(h)
    
    def get_node(self, key: str) -> str:
        """Return the node responsible for this key."""
        if not self.ring:
            raise Exception("Ring is empty")
        h = self._hash(key)
        # Find first ring position >= h (binary search → O(log N))
        idx = bisect.bisect_right(self.sorted_keys, h)
        if idx == len(self.sorted_keys):
            idx = 0  # wrap around
        return self.ring[self.sorted_keys[idx]]

# Usage
ring = ConsistentHashRing(['node1', 'node2', 'node3'], virtual_nodes=150)

print(ring.get_node('user:123'))  # → 'node2'
print(ring.get_node('user:456'))  # → 'node1'

# Add a node
ring.add_node('node4')
print(ring.get_node('user:123'))  # may now → 'node4' if remapped (only ~25% of keys)
print(ring.get_node('user:456'))  # unchanged if not in node4's range
```

### Replication with Consistent Hashing

For fault tolerance, data is typically replicated to the next N nodes clockwise from the primary:

```
Replication factor R=3:
  Key K1 (hash=50) → primary=A(100), replica1=B(200), replica2=C(300)

If Node A fails:
  B becomes primary for K1 (already has a replica)
  C is still replica1
  New replica is created on the next node after C → D
  K1 is never lost — R=3 tolerates up to 2 simultaneous failures.
```

Used in Cassandra, DynamoDB, Dynamo-style systems.

### Consistent Hashing in Load Balancing

Consistent hashing is also used for **sticky load balancing** — routing requests from the same client or with the same key to the same backend server:

```
Use Case: Distributed cache affinity
  Route all requests for user:123 to the same cache node.
  Without consistent hashing: user:123 hits different cache nodes → cache cold.
  With consistent hashing: user:123 always → cache node B → high hit rate.

Use Case: Two-tier cache architecture
  Layer 1: Consistent hash to L1 cache server (per-client affinity)
  Layer 2: L1 miss → S3/DB
```

---

## Key Trade-offs

| Advantage | Disadvantage |
|---|---|
| Only 1/N keys remapped when adding/removing a node | Without vnodes, distribution can be uneven |
| No central coordinator needed | Hot keys still cause imbalance (need separate mitigation) |
| Works well with dynamic node membership | Vnodes add complexity to failure recovery |
| Enables decentralized key routing in DHTs | Harder to ensure replication across failure domains without rack-awareness |

---

## When to use

Consistent hashing is appropriate for:
- **Distributed caches** (Memcached client-side sharding, Redis Cluster): route requests to the correct shard without a central routing table.
- **Distributed databases** (Cassandra, DynamoDB, Riak): assign key ranges to nodes with minimal data migration on topology changes.
- **Content delivery and load balancing**: route requests with the same key to the same backend.
- **Distributed hash tables (DHTs)**: peer-to-peer systems like BitTorrent, Chord.

---

## Common Pitfalls

- **Too few virtual nodes**: with V=1 per node, the distribution is highly uneven. Use V=100–200 for smooth distribution (~1% standard deviation of load).
- **Hot keys**: even with consistent hashing, a single high-traffic key always maps to one node. Use key-based fan-out, local caching, or add a per-hot-key random suffix with a read-through cache to spread the load.
- **Not considering replication topology with failure zones**: with R=3, ensure the 3 replicas are on nodes in different racks/AZs. Rack-aware placement (Cassandra's NetworkTopologyStrategy) must be configured explicitly on top of consistent hashing.
- **Ring watch divergence**: in a fully decentralized system, different nodes may have slightly different ring views (gossip propagation delay). Brief inconsistencies in routing are normal and are corrected by replication + read-repair mechanisms.

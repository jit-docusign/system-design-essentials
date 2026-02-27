# Sharding

## What it is

**Sharding** (also called **horizontal partitioning**) is the practice of splitting a large dataset across **multiple independent database nodes**, where each node (shard) holds a distinct subset of the data. Unlike replication (which copies the same data), sharding divides the data so that each write and read is handled by exactly one shard.

Sharding allows write throughput to scale beyond what a single database server can handle and distributes storage across many nodes.

---

## How it works

### Core Concept

Instead of storing all 1 billion users in one `users` table on one server, sharding splits them across, say, 10 servers — each holding ~100 million users:

```
User ID 1–100M   → Shard 1 (db-shard-01)
User ID 100M–200M → Shard 2 (db-shard-02)
...
User ID 900M–1B  → Shard 10 (db-shard-10)
```

The application (or a routing layer) determines which shard to use before executing any query based on the **shard key**.

### Shard Key Selection

The **shard key** is the column (or derived value) used to determine which shard a given record belongs to. It is the most critical decision in a sharding design.

**Properties of a good shard key:**
- **High cardinality**: enough distinct values to distribute evenly.
- **Even distribution**: avoids hot spots (one shard receiving disproportionately more traffic).
- **Query locality**: queries that often run together should target the same shard to avoid cross-shard queries.
- **Immutability**: changing the shard key value of a record requires migrating it to a different shard — avoid mutable shard keys.

Common shard keys: `user_id`, `tenant_id`, `order_id`.

### Range-Based Sharding

Records are assigned to shards based on ranges of the shard key:

```
user_id 1–1,000,000     → Shard A
user_id 1,000,001–2,000,000 → Shard B
...
```

- **Pros**: Simple to reason about; supports efficient range scans within a shard; easy to identify which shard a record belongs to.
- **Cons**: **Hot spots** if recent data is accessed disproportionately (e.g., new users sign up into the last shard); uneven data growth over time.

Use case: time-series data partitioned by date range, customer IDs with relatively uniform distribution.

### Hash-Based Sharding

The shard is determined by computing `hash(shard_key) % num_shards`:

```
shard_number = MD5(user_id) % 10
```

- **Pros**: Distributes data evenly across shards by default.
- **Cons**: **Range queries become cross-shard** (data is scattered). Adding or removing shards requires **resharding** — rehashing all data. 

Use case: write-heavy workloads where even distribution matters more than range query locality.

### Consistent Hashing

A technique to minimize data movement when the number of shards changes. Nodes are placed on a hash ring, and each key maps to the nearest node clockwise on the ring.

```
         Hash Ring (0–359 degrees)
               0°
         Node A (at 60°)
    Node D         Node B (at 120°)
         Node C (at 240°)

key → hash(key) → find nearest node clockwise
```

When a new node is added, only keys between the previous node and the new node need to move — on average `K/n` keys (where K = total keys, n = number of nodes). Without consistent hashing, all keys would have to be rehashed.

See also: [Load Balancing Algorithms — Consistent Hashing](../traffic/load-balancing-algorithms.md).

### Directory-Based Sharding

A lookup service (shard map / routing table) explicitly maps each shard key range or key value to a specific shard. The routing layer consults this directory for every request.

```
Shard Map:
  tenant_id 1001 → shard-3
  tenant_id 1002 → shard-7
  ...
```

- **Pros**: Flexible — can reassign records to any shard without following a fixed formula; can easily accommodate shards of different sizes.
- **Cons**: The shard map itself is a bottleneck and single point of failure; adds a lookup hop to every request.

### Cross-Shard Queries

Queries that filter or aggregate across multiple shards (or that join across shards) are the main complexity of sharding:

```sql
-- Without sharding: trivial
SELECT COUNT(*) FROM orders WHERE status = 'pending';

-- With sharding: must fan out to all shards, then aggregate
-- Shard 1: SELECT COUNT(*) FROM orders WHERE status = 'pending' → 1,200
-- Shard 2: SELECT COUNT(*) FROM orders WHERE status = 'pending' → 980
-- ...
-- Application layer: SUM(results) = total
```

Strategies for handling cross-shard queries:
- **Scatter-gather**: send query to all shards in parallel, aggregate results in the application layer.
- **Denormalization + duplication**: store derived aggregates or copies of data on one shard to avoid scatter-gather.
- **Dedicated analytics replica**: run cross-shard queries against a read replica that aggregates all shards (data warehouse pattern).

### Resharding

When data outgrows the current shard count, data must be redistributed across more shards. This is disruptive:
1. New shards are added.
2. Data is migrated from old shard ranges to new ones.
3. During migration, both old and new shards may be hot.

Approaches:
- **Double writing**: write to both old and new shard during migration, then cut over reads.
- **Consistent hashing**: minimizes data movement on rebalancing.
- **Virtual shards**: use more virtual shards than physical nodes; when you add a node, only move virtual shards to it.

---

## Key Trade-offs

| Advantage | Disadvantage |
|---|---|
| Write throughput scales linearly with shard count | Cross-shard transactions are complex or impossible |
| Each shard is smaller → faster index scans and vacuums | Cross-shard queries require scatter-gather |
| Failure blast radius reduced (one shard, not all data) | Resharding is operationally expensive |
| Holds arbitrarily large datasets | Application must understand sharding logic |
| Strong data isolation (e.g., tenant-level sharding) | Schema changes must be applied to all shards |

---

## When to use

Sharding is warranted when:
- A single database can no longer keep up with **write throughput** even after vertical scaling and query optimization.
- The **dataset is too large** to fit on a single server's storage.
- You need strong data isolation between tenants (each tenant on a separate shard).

Sharding adds immense operational complexity. **Delay sharding as long as possible.** Exhaust vertical scaling, read replicas, caching, and query optimization first.

---

## Common Pitfalls

- **Wrong shard key causing hot spots**: sharding on `created_at` or auto-increment ID causes all recent writes to land on one shard while older shards are idle.
- **Sharding too early**: adding sharding before it's needed adds complexity without benefit. Shard only when a single node is genuinely the bottleneck.
- **Cross-shard transactions**: if your application requires atomic updates across two shards, you need distributed transactions (two-phase commit), which are slow, complex, and prone to failure. Redesign data layout to keep related data on the same shard.
- **Forgetting to shard secondary tables**: if `orders` is sharded by `user_id`, `order_items` should also be sharded by `user_id` so a user's order data stays on the same shard.
- **Not planning for resharding**: your sharding strategy must account for future growth. Design with consistent hashing or virtual shards from the start.

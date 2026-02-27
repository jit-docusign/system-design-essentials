# Memcached

## What it is

**Memcached** is a high-performance, distributed, **in-memory key-value cache**. It is simpler and more focused than Redis — storing only opaque byte values by string key with no persistence, no replication, no data structures beyond strings/bytes, and no built-in clustering.

Memcached trades Redis's feature richness for simplicity and raw throughput. At Facebook, Memcached serves hundreds of millions of operations per second across thousands of nodes.

---

## How it works

### Data Model

```
SET <key> <value> <exptime> <bytes>
GET <key>
DELETE <key>
INCR <key> <amount>   (atomic increment)
CAS  <key> <value> <cas_token>  (check-and-set / optimistic update)
```

Values are opaque byte strings up to 1MB. Memcached has no awareness of JSON, objects, or any structure within the value. The application serializes/deserializes.

```python
client.set("user:1001", json.dumps(user_data), expire=300)
raw = client.get("user:1001")
user_data = json.loads(raw) if raw else None
```

### Architecture

- **No primary-replica**: Memcached is intentionally masterless — each node is independent.
- **No built-in sharding**: clients distribute keys across the pool using client-side consistent hashing.
- **Multi-threaded**: unlike Redis's single-threaded model, Memcached uses multiple threads to handle concurrent connections — better CPU utilization on multi-core machines.

```
        Client-side consistent hashing
Pool:   [Node A]  [Node B]  [Node C]  [Node D]

hash("user:1001") → Node B
hash("user:1002") → Node A
hash("session:x") → Node D
```

If a node fails, keys that hashed to it are simply lost — no failover, no replication. This is acceptable for a cache (data is rebuilt on miss).

### Memory Model: Slab Allocator

Memcached uses a **slab allocator** to avoid memory fragmentation. Memory is pre-divided into size classes (slabs), and each value is stored in the appropriate slab class for its size:

```
Slab class 1:  chunks of 96 bytes   (small values)
Slab class 2:  chunks of 120 bytes
...
Slab class 42: chunks of 1,048,576 bytes (1MB max)
```

When a slab class is full, Memcached evicts an LRU item from that class to make room. Slab imbalance (all memory allocated to one size class when another size class is needed) can cause premature eviction.

### SASL Authentication

Memcached supports SASL-based authentication for networks where untrusted clients could connect. In many deployments (trusted LAN, same VPC), it runs unauthenticated.

### Memcached vs Redis Summary

| Feature | Memcached | Redis |
|---|---|---|
| Data structures | String/bytes only | 10+ types (string, hash, list, set, sorted set, stream, …) |
| Threading | Multi-threaded | Single-threaded (I/O threads in Redis 6+) |
| Persistence | None | RDB, AOF |
| Replication | None built-in | Async primary-replica |
| Clustering | Client-side sharding | Redis Cluster (server-side) |
| Pub/Sub | No | Yes |
| Scripting | No | Lua scripts |
| Max value size | 1MB | 512MB |
| Simplicity | Very simple | More complex |

**When to choose Memcached over Redis:**
- You need pure, maximum-throughput caching with no other features.
- You want the simplicity of a single purpose with no operational footprint from persistence, replication, or clusters.
- Your team is comfortable managing client-side sharding.
- You need multi-threaded performance for very high connection counts on multi-core machines.

**When to choose Redis over Memcached:**
- You need persistence, replication, or automatic failover.
- You need richer data structures (sorted sets for leaderboards, streams for messaging).
- You need pub/sub, distributed locks, or atomic operations on complex structures.
- You want a single tool that handles caching + sessions + queues + rate limiting.

**Practical reality**: Redis has largely replaced Memcached in new systems because the richer feature set eliminates the need for additional infrastructure. Memcached is still maintained and used where its simplicity is preferred, but new greenfield systems typically default to Redis.

---

## Key Trade-offs

| Advantage | Disadvantage |
|---|---|
| Very simple operational model | No persistence — data lost on restart |
| Multi-threaded (better multi-core utilization for pure cache) | No replication or built-in HA |
| Slab allocator avoids fragmentation | Only string values — no rich data structures |
| Client-side sharding is trivially horizontal | Client must handle node failure/resharding |
| Proven at massive scale (Facebook, Twitter) | Feature-poor vs Redis |

---

## When to use

Memcached makes sense when:
- Your use case is **pure caching** with no need for persistence, rich data structures, or HA failover.
- You want **operational simplicity** — no replication setup, no cluster management.
- You're running on a **multi-core machine** and want to use all cores for cache throughput (Memcached is multi-threaded; Redis effectively single-threaded per shard).
- You're maintaining an **existing Memcached deployment** and the migration cost to Redis isn't justified.

---

## Common Pitfalls

- **Slab allocation imbalance**: if your value sizes shift significantly over time (e.g., now caching large HTML fragments instead of small JSON objects), Memcached may behave suboptimally. Monitor per-slab-class hit rates.
- **No replication = no graceful failover**: a failed Memcached node means cache miss for all keys that were on that node. Pre-warm replacement nodes if possible, or use consistent hashing in the client to minimize the blast radius.
- **Assuming cache is durable**: Memcached is not persistent. If your application assumes the cache will survive a restart (e.g., storing session state only in Memcached without a persistent fallback), users will be logged out on every restart.
- **Not handling `cas` (check-and-set) misses**: for read-modify-write operations, use CAS to prevent lost updates. Failing to handle the CAS failure case leads to race conditions.

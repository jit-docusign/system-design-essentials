# Distributed Cache

## What it is

A **distributed cache** is a caching layer spread across multiple nodes, allowing the cache capacity and request throughput to scale beyond a single machine. Instead of one cache server, a cluster of cache nodes collectively stores data - each node holding a subset of the total cached data.

Distributed caching is the architecture used by large-scale systems (Facebook's Memcache, Twitter's Twemcache, Netflix's EVCache) to serve billions of cache operations per day.

---

## How it works

### Single Node vs Distributed Cache

```
Single cache node:
  App → [Redis / Memcached] → DB
  Limit: one server's RAM (~128–256GB), one server's CPU/network

Distributed cache:
  App → [Node 1 | Node 2 | Node 3 | ... Node N] → DB
  Scale: multiply capacity by adding nodes
```

### Data Distribution: Consistent Hashing

The key question in a distributed cache: **which node stores a given key?**

**Modulo hashing** (naive):
```
node = hash(key) % num_nodes
```
Problem: adding or removing a node changes the mapping for `(N-1)/N` of all keys → massive cache miss tsunami on any cluster size change.

**Consistent hashing** (production-standard):
Nodes are placed on a virtual ring. Each key maps to the first node clockwise from `hash(key)` on the ring.

```
        Ring (hash space 0 to 2^32)
          Node A (at 60°)
Node D                      Node B (at 120°)
          Node C (at 240°)

Key "user:1001" → hash → 90° → maps to Node B
Key "user:2002" → hash → 210° → maps to Node C
```

When a new node is added: only keys between the new node and the previous clockwise node need to move. All other keys stay put.

**Virtual nodes (vnodes)**: each physical node is represented by multiple virtual points on the ring (100–200 per node). This ensures more even key distribution and makes rebalancing more granular.

### Replication in Distributed Cache

**No replication (pure partitioned cache)**:
- Each key lives on exactly one node.
- Node failure causes cache miss for all keys on that node (thundering herd risk).
- Simplest approach; acceptable if DB can absorb temporary miss load.

**Replication factor > 1**:
- Each key is stored on N consecutive nodes on the ring.
- Node failure degrades read hit rate but doesn't cause 100% miss for affected keys.
- Redis Cluster uses 1 primary + N replicas per shard.

### Hotspot Mitigation

Some keys are accessed orders of magnitude more than others (celebrity posts, product launches, breaking news). Hot keys create load imbalance — one node handles a disproportionate share of requests.

**Strategies:**

1. **Local, in-process cache (L1 cache)**: each application server maintains a small in-process cache (LRU map in memory). Super-hot keys are served from process memory without any network call.
   ```
   L1 (in-process, ~1000 items, 1s TTL)
     → Hit: return immediately (no network)
     → Miss: fetch from L2 (distributed cache)
       → Miss: fetch from DB
   ```

2. **Key replication with random suffix**: for read-heavy hot keys, store N copies: `key:0`, `key:1`, ... `key:N-1`. On read, randomly pick a copy. Writes must update all copies.
   ```
   shard = random.randint(0, 9)
   value = cache.get(f"trending_posts:{shard}")
   ```

3. **Read replication**: distributed cache with read replicas for hot shards.

### Distributed Lock Pattern

Redis's distributed cache can serve as a distributed lock coordinator:
```
node1 = redis_node_at(hash("lock:order-123"))
acquired = node1.set("lock:order-123", lock_id, NX=True, EX=30)
```

See [Distributed Locks](../distributed/distributed-locks.md) for the full RedLock algorithm.

### Cache Cluster Management Events

**Node failure**: consistent hashing re-routes to the next node; affected data is a cold miss and refilled from DB.

**Node addition (scaling out)**: a fraction of keys are remapped to the new node; others stay on their current nodes. Traffic is gradually absorbed by the new node as its cache warms.

**Node removal (scaling in)**: keys on the removed node become cold misses. This is expected; DB absorbs the miss load temporarily.

### Facebook's Memcache Architecture

Facebook's landmark 2013 paper "Scaling Memcache at Facebook" described their architecture for handling 1 billion+ operations/second:

1. **Regional pools**: Memcache clusters within a data center region.
2. **Cold cluster warm-up**: new cache clusters are warmed by requesting data from a warm cluster rather than hitting DB directly.
3. **Leases**: prevents thundering herd and stale set races — cache issues a token on miss; client must present token to write, ensuring freshness.
4. **Mcrouter**: a proxy layer that routes cache requests, multiplexes connections, and provides geographic routing.

---

## Key Trade-offs

| Advantage | Disadvantage |
|---|---|
| Horizontal scale — add nodes for more capacity | Complexity of key routing and cluster management |
| Fault isolation — one node failure ≠ total cache loss | Network hop per cache operation (vs in-process cache) |
| Independent scaling from application tier | Hot key imbalance — some nodes more loaded than others |
| Supports very large working sets (TB of cache) | Replication adds write overhead |

---

## When to use

- Your single cache node is hitting RAM or CPU limits.
- Your working set (hot data) is larger than one machine's RAM.
- You need fault tolerance — one cache node failure should not cascade to DB.
- You're operating at a scale where even a single Redis/Memcached node becomes a SPOF.

---

## Common Pitfalls

- **Thundering herd after node failure**: a failed node in a large cluster causes all its keys to be cache misses simultaneously. Rate-limit cache miss DB fetches, implement short-circuit locking.
- **Not accounting for network latency**: a distributed cache adds network hops. For ultra-latency-sensitive paths, complement with an L1 in-process cache.
- **Modulo hashing and resharding pain**: using `hash % N` instead of consistent hashing makes adding/removing nodes catastrophic.
- **No monitoring on cache hit rates per node**: hot key imbalance is invisible without per-node hit rate and key-level access counters. Monitor closely.

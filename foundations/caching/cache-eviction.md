# Cache Eviction Policies

## What it is

A **cache eviction policy** determines which cached item to remove when the cache runs out of space and a new item needs to be stored. Since caches are bounded in size (by RAM), eviction is inevitable in any long-running system.

The right eviction policy ensures the cache retains the **most useful items** — maximizing cache hit rate — while discarding data that is unlikely to be accessed again.

---

## How it works

### Why Eviction Matters

Every eviction of a "still-useful" item causes a cache miss on its next access, sending a request to the origin DB. Choosing the wrong eviction policy can result in cache thrashing — items being evicted and immediately needed again — eliminating most cache benefit.

### Common Eviction Algorithms

#### LRU — Least Recently Used

Evicts the item that was **accessed (read or written) least recently**. Based on the assumption that recently used data is more likely to be used again.

```
Cache state (most recent → least recent):
  [D] [B] [A] [C] [E]

Cache FULL + add new item [F]:
→ Evict [E] (least recently used)
→ New state: [F] [D] [B] [A] [C]
```

**Implementation**: usually a doubly-linked list + hash map. On each access, the item is moved to the head of the list. On eviction, remove from the tail.

- **Pros**: intuitive; performs well for most workloads.
- **Cons**: vulnerable to a "cache scan" where a large sequential read evicts all hot items. A single full-table scan can wipe the entire LRU cache.

Used by: Redis (`allkeys-lru`, `volatile-lru`), most in-process caches.

#### LFU — Least Frequently Used

Evicts the item that was **accessed least frequently** over a time window.

```
Item access counts:
  A: 450 hits
  B: 2 hits
  C: 800 hits
  D: 1 hit
  E: 90 hits

Cache FULL → evict D (lowest frequency: 1 hit)
```

- **Pros**: better than LRU when access frequency is more predictive of future access than recency. Good for skewed workloads where a small set of items accounts for most traffic.
- **Cons**: items that were hot long ago but are no longer relevant stay in the cache (they've accumulated high counts). Frequency counts can decay over time to mitigate this (LFU with decay — "TinyLFU").

Used by: Redis (`allkeys-lfu`, `volatile-lfu`). Caffeine (Java in-process cache) uses TinyLFU.

#### Random Eviction

Evicts a randomly selected item.

- **Pros**: O(1), extremely simple to implement.
- **Cons**: poor cache efficiency — may evict frequently-accessed items.

Used in: CPU hardware caches; rarely used in application-level caches.

#### FIFO — First In, First Out

Evicts the item that was **inserted earliest**, regardless of access pattern.

- **Pros**: simple.
- **Cons**: doesn't consider access frequency or recency. Evicts old but frequently-used items.

#### MRU — Most Recently Used

Evicts the **most recently used** item first. This inverts LRU.

- **When useful**: workloads where once an item is read it won't be read again soon (e.g., scanning through a large dataset sequentially). Evicting MRU avoids filling the cache with once-read scan items.

#### CLOCK (Second-Chance)

An approximation of LRU with lower overhead. Each item has a reference bit. Instead of a sorted list, a clock hand sweeps through items:
- Bit = 1 → clear bit and move to next item (second chance).
- Bit = 0 → evict this item.

On access, set the item's bit to 1.

Used in: operating system page replacement algorithms.

### Redis Eviction Policies

Redis supports configuring the eviction policy globally:

| Policy | Description |
|---|---|
| `noeviction` | Return error on writes when memory is full |
| `allkeys-lru` | LRU across all keys |
| `volatile-lru` | LRU only among keys with TTL set |
| `allkeys-lfu` | LFU across all keys |
| `volatile-lfu` | LFU only among keys with TTL |
| `allkeys-random` | Random eviction across all keys |
| `volatile-random` | Random eviction only among TTL keys |
| `volatile-ttl` | Evict key with the nearest TTL expiry |

Configure with:
```
maxmemory 4gb
maxmemory-policy allkeys-lru
```

**Rule of thumb**:
- Use `allkeys-lru` for a pure cache where all data is expendable.
- Use `volatile-lru` if some data must never be evicted (set those without TTL).
- Use `allkeys-lfu` for workloads with a very hot set of keys (Zipfian distribution).

### Cache Size — Working Set Sizing

Eviction rate is a function of cache size relative to the working set:

```
Working set = set of distinct items accessed within a time window

If cache size ≥ working set size → very low eviction, high hit rate
If cache size << working set size → frequent eviction, low hit rate
```

The **working set principle**: allocate enough cache to hold the hot working set. Many applications follow a Zipfian distribution — 20% of keys account for 80% of access. Sizing the cache to hold that 20% achieves near-perfect hit rates.

### Hit Rate and Its Impact

```
Cache hit rate = (hits / (hits + misses)) * 100

Example: 100K req/s with 90% hit rate
  → 90K requests served from cache (~0.2ms)
  → 10K requests hit DB (~5ms)
  → Avg latency ≈ 90% × 0.2ms + 10% × 5ms = 0.68ms

At 50% hit rate:
  → 50K request hit DB → DB load 5× higher
  → Avg latency ≈ 50% × 0.2ms + 50% × 5ms = 2.6ms (3.8× worse)
```

---

## Key Trade-offs

| Policy | Best for | Weakness |
|---|---|---|
| LRU | General caching; recency-dominated workloads | Cache pollution from sequential scans |
| LFU | Skewed/Zipfian workloads | Stale high-count items; history pollution |
| FIFO | Simple, predictable workloads | Ignores access patterns |
| MRU | Sequential scan workloads | Counterintuitive; rarely useful |
| Random | Simplicity, approximate fairness | No intelligent selection |

---

## Common Pitfalls

- **Cache size too small for working set**: if the cache can only hold 1% of the hot data, eviction is constant, hit rate is low, and the cache provides little benefit. Right-size the cache to the working set.
- **Not monitoring eviction rate**: high eviction rates are a signal to increase cache size or improve TTL tuning. Add eviction rate to dashboards.
- **LRU and sequential scan workloads**: a full table export or data migration that reads 10M unique keys will evict all frequently-used data from an LRU cache. Use `volatile` policies with TTL to protect important items, or use scan-resistant algorithms.
- **noeviction in production**: using `noeviction` means Redis will return errors on all write operations when full. This crashes many applications. Prefer `allkeys-lru` unless you have very specific protection requirements.

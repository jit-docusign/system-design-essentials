# Caching Strategies

## What it is

**Caching** is the technique of storing copies of data in a faster, closer storage layer so that future requests for the same data can be served faster — without hitting the slower origin (database, external API, computed result).

A **caching strategy** defines *how* the cache interacts with the primary data store: when data is loaded into the cache, when it's written back to the store, and what happens on a cache miss.

---

## How it works

### 1. Cache-Aside (Lazy Loading)

The application manages the cache explicitly. The cache sits beside the application — not in the data path. This is the most common pattern.

```
READ path:
  1. Application checks cache first.
  2. Cache HIT  → return value.
  3. Cache MISS → fetch from database → write to cache → return value.

WRITE path:
  1. Application writes to database.
  2. Application invalidates (or updates) the cache entry.
```

```python
def get_user(user_id):
    # Try cache first
    cached = redis.get(f"user:{user_id}")
    if cached:
        return json.loads(cached)  # cache HIT

    # Cache MISS: fetch from DB
    user = db.query("SELECT * FROM users WHERE id = ?", user_id)
    redis.setex(f"user:{user_id}", 300, json.dumps(user))  # cache 5 min
    return user
```

**Characteristics:**
- Only data that is actually requested gets cached (lazy = on-demand).
- Cache starts cold on startup — first requests are slow (thundering herd risk).
- Stale data possible if DB is updated without invalidating cache.
- Cache failures don't break the system (falls through to DB).

**Best for**: most general-purpose caching; read-heavy workloads where not all data is equally hot.

### 2. Read-Through

Similar to cache-aside but the cache library/layer handles the miss automatically — the application always reads from the cache, and the cache fetches from the database on a miss.

```
App → Cache
          MISS → Cache fetches from DB → Cache stores → returns to App
          HIT  → Returns from cache
```

**Difference from cache-aside**: the cache — not the application — contains the DB fetch logic. This centralizes caching logic at the cost of coupling the cache layer to the data source.

**Best for**: caching proxies and middleware; when you want to abstract caching from application code.

### 3. Write-Through

Every write to the database also writes to the cache synchronously. The cache is always kept up to date.

```
App writes → Cache (write) → Database (write)
Both succeed → return OK

Cache failure → write fails (or write-through is bypassed)
```

**Pros**: cache is never stale — reads always hit warm, fresh cache.  
**Cons**: write latency increases (two writes per operation); cache may hold data that is never read (wasted storage).

**Best for**: data that is frequently written AND frequently read; systems where stale reads are unacceptable.

### 4. Write-Behind (Write-Back)

The application writes to the cache, and the cache asynchronously flushes to the database later (batching writes for efficiency).

```
App writes → Cache (write) → returns OK immediately
                              ↓ (async, batched)
                            Database (write)
```

**Pros**: write latency is extremely low (write to memory only); writes can be batched (efficient for bulk operations).  
**Cons**: risk of data loss if the cache crashes before flushing to DB; complexity in ensuring all writes propagate.

**Best for**: high-frequency write scenarios where write batching improves throughput — counters, analytics events, gaming scores.

### 5. Refresh-Ahead (Prefetching)

The cache proactively refreshes cached entries *before* they expire, based on predicted access patterns or approaching TTL expiry.

```
Cache entry: user:1001
  TTL = 6 minutes remaining
  Prefetch threshold: when TTL < 1 minute

If TTL < 1 minute AND recent access → background refresh
→ new value loaded, TTL reset
→ next request hits warm cache (no stall)
```

**Pros**: eliminates cache miss latency for hot, frequently-accessed items.  
**Cons**: may refresh items that won't be accessed again (wasted DB queries); complexity.

**Best for**: predictable hot-key patterns; cache entries with expensive DB queries that must stay warm.

---

## Comparison Table

| Strategy | Read path | Write path | Staleness risk | Complexity |
|---|---|---|---|---|
| Cache-aside | App checks cache → DB on miss | Invalidate on write | Yes (between write and invalidation) | Low |
| Read-through | Cache fetches from DB on miss | Invalidate or write-through | Yes | Medium |
| Write-through | Cache returns data | Write to cache + DB atomically | Minimal | Medium |
| Write-behind | Cache returns data | Write to cache; async to DB | Yes (crash risk) | High |
| Refresh-ahead | Cache returns warm data | Varies | Minimal for hot keys | High |

---

## Key Trade-offs

- **Consistency vs performance**: write-through maximizes freshness but adds write latency. Cache-aside may serve stale data between write and invalidation.
- **Complexity vs control**: cache-aside gives the application full control. Read-through/write-through abstract caching but couple the cache layer to the database.
- **Warmth**: lazy-loaded caches start cold. Pre-warming (loading data into cache at startup) avoids initial thundering herd.

---

## When to use

| Workload | Recommended strategy |
|---|---|
| Read-heavy, writes rare | Cache-aside with TTL |
| Frequent reads + frequent writes | Write-through |
| Very high write rate, can tolerate async flush | Write-behind |
| Predictably hot items, must avoid miss stall | Refresh-ahead |
| Caching layer managed by infrastructure (not app) | Read-through |

---

## Common Pitfalls

- **Thundering herd on cache miss**: when a popular cached item expires, many concurrent requests all miss and hammer the DB simultaneously. Use mutex / lock on first miss ("dog-pile protection"), or refresh-ahead to prevent expiry.
- **Not setting TTLs**: items without TTL accumulate in the cache indefinitely. Set appropriate TTLs based on how frequently data changes.
- **Inconsistent invalidation**: data is updated in the DB but the cache invalidation is missed due to a code path that bypasses cache. Centralize write paths that touch cached data.
- **Caching without understanding access patterns**: caching everything doesn't help if the cache thrashes (all cache space consumed by objects that are accessed only once). Only cache frequently accessed, expensive-to-compute data.

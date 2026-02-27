# Application-Level Caching

## What it is

**Application-level caching** refers to the caching strategies and layers managed within the application tier itself — as opposed to infrastructure-level caches (CDN, database buffer pool, OS page cache). It includes in-process (in-memory) caches, external remote caches (Redis/Memcached), and query result caches within the application code.

Understanding where to cache in the application stack — and at which level — is as important as knowing which caching technology to use.

---

## How it works

### The Caching Hierarchy

Multiple caching layers exist in a typical web application. Each is faster, smaller, and closer to the requester than the layer below it:

```
L1 (fastest):  Browser cache / CDN edge cache
               ↓ (miss)
L2:            Load balancer / reverse proxy cache (Nginx, Varnish)
               ↓ (miss)
L3:            Application process memory (in-process cache)
               ↓ (miss)
L4:            Distributed cache (Redis, Memcached)
               ↓ (miss)
L5:            Database buffer pool (InnoDB buffer, PostgreSQL shared buffers)
               ↓ (miss)
L6 (slowest):  Disk / Storage
```

Requests ideally resolve at the highest (fastest) level. Application-level caching spans L3 (in-process) and L4 (distributed cache).

### L3: In-Process Cache (Local Memory Cache)

An in-process cache stores data directly in the application server's heap memory. There is zero network overhead — access is a memory read.

**Advantages:**
- Sub-microsecond access (vs 0.5–1ms for Redis, 1–10ms for DB).
- No network hop.
- No serialization cost.

**Disadvantages:**
- Small capacity — bounded by server JVM heap or process RAM.
- Not shared between application instances — each server has its own cache (inconsistency risk).
- Data lost when process restarts.

**Common implementations:**

*Java*: Google Guava `LoadingCache` or Caffeine (high-performance, TinyLFU eviction):
```java
LoadingCache<Long, User> userCache = Caffeine.newBuilder()
    .maximumSize(10_000)
    .expireAfterWrite(5, TimeUnit.MINUTES)
    .build(userId -> dbService.getUser(userId));

User user = userCache.get(userId);  // auto-loads on miss
```

*Python*: `functools.lru_cache` or `cachetools`:
```python
from cachetools import TTLCache, cached

cache = TTLCache(maxsize=1000, ttl=300)  # 1000 items, 5-minute TTL

@cached(cache)
def get_config(config_key: str) -> str:
    return db.fetch_config(config_key)
```

*Node.js*: `node-cache` or a simple `Map`:
```javascript
const cache = new Map();
function getUser(id) {
    if (cache.has(id)) return cache.get(id);
    const user = db.findUser(id);
    cache.set(id, user);
    setTimeout(() => cache.delete(id), 300_000); // TTL: 5 minutes
    return user;
}
```

### L3+L4: Two-Level Cache Pattern

A common pattern combines in-process (L1/L3) and distributed (L4) caches:

```
Request → Check in-process cache (L1)
            → HIT: return (nanoseconds)
            → MISS: check Redis (L2)
                → HIT: populate in-process cache → return (milliseconds)
                → MISS: fetch from DB → populate Redis + in-process cache → return (tens of ms)
```

**Benefits**: popular items served from local memory; Redis serves as distributed shared layer; DB is protected by both layers.

**Challenge**: consistency across application instances — when data changes, each instance's local cache must be invalidated. Solutions:
- Short TTL on in-process cache (accept brief staleness).
- Pub/Sub invalidation: Redis pub/sub publishes invalidation events; each app server subscribes and clears its local cache entry.
- Local cache only for truly immutable data (feature flags, configuration, static lookups).

### Query Result Caching

Expensive queries are cached at the application layer:

```python
def get_user_stats(user_id):
    cache_key = f"user_stats:{user_id}"
    result = redis.get(cache_key)
    if result:
        return json.loads(result)

    # Expensive aggregation query
    stats = db.query("""
        SELECT
            COUNT(orders.id) AS total_orders,
            SUM(orders.total) AS lifetime_value,
            MAX(orders.created_at) AS last_order_date
        FROM users
        LEFT JOIN orders ON orders.user_id = users.id
        WHERE users.id = ?
        GROUP BY users.id
    """, user_id)

    redis.setex(cache_key, 600, json.dumps(stats))  # 10-minute TTL
    return stats
```

### Fragment / Object Caching

Different granularities of caching within the application:

| Granularity | What's cached | Example |
|---|---|---|
| Full page | Entire HTTP response | Static marketing pages |
| Page fragment | Rendered component | Sidebar, navigation, featured products |
| Object | Deserialized DB object | User record, config object |
| Query result | Raw DB result set | Aggregation query results |
| Computed value | Business logic output | User risk score, recommendation list |

**Key insight**: cache at the highest level that is cache-safe (consistent enough for callers) and provides the most latency reduction. Caching a full rendered page fragment saves template rendering + DB queries; caching a raw DB row still requires rendering.

### HTTP Caching Headers (Application-Controlled)

The application controls browser and CDN caching behavior via response headers:

```http
# Allow caching for 1 hour in browser + CDN
Cache-Control: public, max-age=3600

# Server-only cache; browsers don't cache
Cache-Control: private, max-age=300

# Always validate with server before using cached copy
Cache-Control: no-cache

# ETag for conditional requests (304 Not Modified)
ETag: "abc123def456"

# Last-Modified for conditional requests
Last-Modified: Wed, 15 Jan 2024 10:00:00 GMT
```

See [CDN](../traffic/cdn.md) for the full HTTP caching treatment.

### Cache-Warming Strategies

A cold cache on deployment or restart causes a thundering herd. Pre-warming strategies:

1. **Scheduled warm-up at startup**: application reads expected hot keys from DB before serving traffic.
2. **Cache shadowing**: before cutting over to a new deployment, warm its cache from a production traffic replay.
3. **Gradual rollout**: roll out new instances slowly; each instance warms gradually as it receives real traffic.
4. **Persistent cache backup**: snapshot Redis to disk and restore on startup (RDB snapshot).

---

## Key Trade-offs

| Approach | Latency | Consistency | Capacity | Sharing |
|---|---|---|---|---|
| In-process cache | Nanoseconds | Per-instance only | Bounded by heap | No (per process) |
| Distributed cache | Milliseconds | Shared across instances | Large (RAM of cluster) | Yes |
| Two-level (L1+L2) | Mixed | Short staleness window in L1 | Large | Yes (via L2) |
| DB query cache | DB I/O savings | Depends on invalidation | Distributed | Yes |

---

## Common Pitfalls

- **Caching mutable per-user data in a shared cache without isolation**: `cache.set("current_user", user_data)` in a shared context would serve the same user to all requests. Always scope cache keys to the user/tenant/context: `f"user:{user_id}:profile"`.
- **In-process cache inconsistency across instances**: if User A's profile is updated by Instance 1, Instance 2's local cache still has the old value. Use short TTLs or pub/sub invalidation.
- **Over-caching volatile data**: caching data that changes every second (live price, inventory count) requires near-real-time invalidation. The overhead may exceed the benefit.
- **Caching exceptions / error responses**: don't cache DB errors or empty results meant to be temporary. A buggy cache entry serving "user not found" for 10 minutes is worse than a cache miss.
- **Not monitoring cache hit rates per cache level**: without instrumentation, you can't tell if the in-process cache is providing value. Add hit/miss counters per cache level.

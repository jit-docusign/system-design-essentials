# Cache Invalidation

## What it is

**Cache invalidation** is the process of removing or updating stale entries in a cache when the underlying source data changes. It is arguably the hardest problem in distributed caching — the phrase "there are only two hard things in computer science: cache invalidation and naming things" (Phil Karlton) has become canonical precisely because getting invalidation right is subtle and error-prone.

The fundamental challenge: the cache must stay synchronized with the source of truth, but invalidating too aggressively destroys cache performance, while invalidating too lazily serves stale data.

---

## How it works

### Time-To-Live (TTL) Based Invalidation

The simplest strategy: every cached entry has a TTL. When it expires, the next read fetches fresh data from the source.

```
Cache entry: user:1001 → {name: "Alice", ...}  TTL: 5 minutes
After 5 minutes → entry expires → next read is a cache miss → DB fetch → re-cache
```

**Pros**: simple, no coordination between cache and DB required.  
**Cons**: data is stale until TTL expiry. TTL too long → stale data. TTL too short → low cache hit rate, high DB load.

**Setting TTL wisely**: TTL should reflect **how often the data changes** and **how much staleness the application can tolerate**:

| Data type | Change frequency | Appropriate TTL |
|---|---|---|
| Static product details | Rarely | Hours to days |
| User profile | Occasionally | Minutes to hours |
| Session data | Every request | Minutes (match session timeout) |
| Real-time inventory | Frequent | Seconds |
| Live sport scores | Constant | 5–30 seconds |
| Authentication state | Rarely but critical | Short TTL + event invalidation |

### Active Invalidation (Explicit Deletion)

When data is updated, the application immediately deletes or updates the corresponding cache entry:

```python
def update_user_profile(user_id, new_name):
    db.execute("UPDATE users SET name = ? WHERE id = ?", new_name, user_id)
    redis.delete(f"user:{user_id}")  # invalidate cache immediately
```

The next read will miss cache and fetch fresh data. This is **reactive invalidation** — invalidate *after* writing.

**Delete vs update on write:**
- **Delete**: simpler; cache miss refills on next read; avoids race conditions where the DELETE races with a concurrent write.
- **Update (write-through)**: cache is kept warm, no miss on next read; requires coordinating the cache update and DB write.

**Recommendation**: default to **delete** on write; move to update only if the cache miss on first post-write read is a measurable problem.

### Event-Driven Invalidation (CDC — Change Data Capture)

For systems where multiple services or cache nodes need to react to database changes, **CDC** propagates changes from the database to the cache automatically:

```
Database (MySQL/PostgreSQL) → CDC tool (Debezium) → Kafka → Cache Invalidator service
                                                              │
                                                              └─> redis.delete(key)
```

The CDC tool reads the database's transaction log (binlog / WAL) and emits events for every INSERT/UPDATE/DELETE. Cache invalidators consume these events and evict affected entries.

This decouples the cache invalidation from the application write path — no risk of forgetting to invalidate from a code path that updates the DB.

**Used by**: Facebook (as described in their Memcache@Facebook paper), LinkedIn, Netflix.

### Cache Invalidation Patterns

#### Pattern 1: Invalidate on Write

```
Write to DB → Invalidate cache → (next read refills cache)
```
Simple and correct. Risk: short window of stale data between write and invalidation if they're not atomic.

#### Pattern 2: Versioned Cache Keys

Instead of invalidating, change the cache key version on every write. Old keys naturally become unreferenced and expire via TTL.

```
Key: user:1001:v5  → {"name": "Alice"}
After update:
Key: user:1001:v6  → {"name": "Alice Smith"}  ← new version
Old key user:1001:v5 expires after TTL
```

Version can be a monotonic counter stored in a "versions" hash:
```
redis.hincrby("versions:user", "1001", 1)
version = redis.hget("versions:user", "1001")
key = f"user:1001:v{version}"
```

**Pros**: no explicit invalidation needed; atomic version increment + read is simpler.  
**Cons**: key space grows until TTL cleanup; requires additional version lookup.

#### Pattern 3: Two-Level Cache Invalidation (Facebook's Lease)

To handle the race condition between a cache miss refill and concurrent writes, use **leases**:
1. On cache miss: client gets a "lease token" instead of the value.
2. Client fetches from DB.
3. Client sets cache value but must present the lease token — if a write happened after the lease was issued, the token is invalidated and the set is rejected.
4. Client retries the read.

This prevents a stale DB value from overwriting a fresher cache entry that was written by a near-concurrent update.

### Race Condition: Stale Set After Invalidation

A subtle bug:

```
Time 0: T1 reads user:1001 → cache miss
Time 1: T2 updates user:1001 in DB → invalidates cache (DELETE)
Time 2: T1 fetches OLD value from DB (reads snapshot before T2's write)
Time 3: T1 sets cache: user:1001 → old value
→ Cache now contains stale data until TTL expires
```

**Mitigations:**
- Short TTL catches eventual recovery.
- Compare-and-set with version number (only set if version matches).
- Lease-based refill (prevents stale set after invalidation).
- Write-through (invalidation is replaced by a cache update — always current).

### Thundering Herd on Invalidation

When a popular cache entry is invalidated (or expires), many concurrent requests all miss simultaneously and flood the database:

```
t=0: cache entry expires for "trending_posts" (used by 10K req/s)
t=0: 1000 concurrent requests all miss
t=0: 1000 concurrent DB queries launched
t=0.2: DB collapses under 1000× normal load
```

**Solutions:**
1. **Short-circuit lock (mutex)**: first request acquires a lock and fetches from DB; others wait or return stale data.
2. **Probabilistic early expiration**: randomly expire the entry before TTL when under load, so it's refreshed proactively in a single request before full expiry.
3. **Jitter on TTL**: add random jitter to TTL values so many entries don't expire simultaneously.
4. **Stale-while-revalidate**: serve the stale cached value while asynchronously refreshing in the background.

---

## Key Trade-offs

| Approach | Freshness | Complexity | Cache hit rate |
|---|---|---|---|
| TTL only | Stale up to TTL | Very low | High |
| Active invalidation | Near-instant | Medium | Medium (misses on write) |
| CDC-based invalidation | Near-instant | High | Medium |
| Versioned keys | Near-instant | Medium | High (old versions expire) |
| Write-through | Always current | Medium | High |

---

## Common Pitfalls

- **Setting TTL too long for rapidly-changing data**: user sees their own profile change not reflected for minutes. Shorten TTL or add explicit invalidation for data the user directly mutates.
- **Forgetting to invalidate from all write code paths**: if user profile can be updated from an admin panel, a mobile API, and a web API, all three must invalidate the cache. Missing one path leads to persistent stale entries.
- **Cache stampede on mass invalidation**: bulk invalidation (e.g., invalidating all cached products after a price migration) causes a simultaneous thundering herd. Invalidate in batches and use per-key TTL jitter.
- **Relying on cache as source of truth**: caches are mirrors. Never write to cache only; always write the source of truth first, then update/invalidate cache.

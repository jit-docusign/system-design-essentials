# Write-Through and Write-Back Caching

## What it is

**Write-through** and **write-back** (also called write-behind) are two cache write strategies that differ in *when* data is written to the primary data store relative to the cache:

- **Write-through**: the cache and the data store are **written simultaneously** on every write operation. The write is not acknowledged until both are updated.
- **Write-back (write-behind)**: data is written to the **cache only** initially. The cache asynchronously flushes changes to the data store later (often batched).

These are write-path complements to the read-path strategies covered in [Caching Strategies](caching-strategies.md).

---

## How it works

### Write-Through

```
Application write request
        │
        ▼
   ┌──────────┐
   │  Cache   │  ← write immediately
   └──────────┘
        │
        ▼ (synchronous)
   ┌──────────┐
   │ Database │  ← also write immediately
   └──────────┘
        │
  ACK returned to client only after BOTH succeed
```

The cache always mirrors the database. Any read can be served from cache without risk of returning stale data.

#### Write-Through Example (Application Code)

```python
def update_product_price(product_id, new_price):
    # Write to DB
    db.execute("UPDATE products SET price = ? WHERE id = ?", new_price, product_id)
    # Write to cache atomically in the same operation
    redis.set(f"product:{product_id}:price", new_price, ex=3600)
    return new_price
```

If the DB write succeeds but the cache write fails, the cache will contain stale data until it expires via TTL. For full consistency, wrap both in a unit of work and handle the failure case (e.g., delete the cache entry so a miss forces a DB fetch).

```python
def update_product_price(product_id, new_price):
    db.execute("UPDATE products SET price = ? WHERE id = ?", new_price, product_id)
    try:
        redis.set(f"product:{product_id}:price", new_price, ex=3600)
    except RedisError:
        redis.delete(f"product:{product_id}:price")  # invalidate on write failure
```

#### Characteristics

| Property | Write-Through |
|---|---|
| Write latency | Slightly higher (two writes per operation) |
| Data freshness | Always consistent — cache == DB |
| Write amplification | Every write goes to both cache and DB |
| Read performance | High cache hit rate — cache is always warm |
| Risk of data loss | None (DB is always updated first) |

**Best for**: data that is read very frequently after being written; user profiles, product details, configuration data.

### Write-Back (Write-Behind)

```
Application write request
        │
        ▼
   ┌──────────┐
   │  Cache   │  ← write immediately → ACK returned to client
   └──────────┘
        │
    (async, after delay or on batch flush)
        │
        ▼
   ┌──────────┐
   │ Database │  ← written later, in batches
   └──────────┘
```

The response to the write is fast (in-memory only). The DB write happens asynchronously, often batched with other pending writes for efficiency.

#### Write-Back Mechanisms

**Dirty bit approach**: cache marks a modified entry as "dirty". A background flush thread periodically scans for dirty entries and writes them to DB.

```
On write:
  cache["product:1:price"] = new_price
  dirty_keys.add("product:1:price")  # mark dirty

Background flush (every 1s or when dirty_keys.size > threshold):
  for key in dirty_keys:
      db.write(key, cache[key])
      dirty_keys.remove(key)
```

**Event queue approach**: writes are appended to a persistent queue (Kafka, SQS); a consumer processes the queue and writes to DB.

```
App write → Cache → Kafka topic (durable)
                       │
               Consumer → DB write (batched)
```

Using a durable queue mitigates data loss risk — even if the cache crashes, the write events are preserved in Kafka.

#### Characteristics

| Property | Write-Back |
|---|---|
| Write latency | Very low (in-memory write only) |
| Data freshness | Risk of stale DB data during async window |
| Write amplification | Low — writes are batched |
| Risk of data loss | Yes — if cache crashes before flush, writes are lost |
| DB write efficiency | High — batched writes reduce I/O operations |

**Best for**: high write throughput scenarios where write batching is beneficial and some data loss is tolerable — metrics, analytics event logging, counters, gaming scores.

### Comparison: Write-Through vs Write-Back

| Dimension | Write-Through | Write-Back |
|---|---|---|
| Write latency | Higher (2 writes, synchronous) | Very low (1 write, async) |
| Data durability | Immediate — DB always current | Deferred — DB may be behind |
| Data loss risk | None | Yes (crash before flush) |
| Write throughput | Limited by DB write speed | Higher (batching) |
| Complexity | Low | Higher (flush logic, failure handling) |
| Cache warmth | Always warm for written data | Always warm for written data |
| Read-after-write | Consistent | Consistent from cache; stale from DB directly |

### Combining with Cache Invalidation

Write-through and write-back are for updating the cache on writes. Cache-aside with invalidation is for removing entries on writes. The choice depends on whether the cache should remain warm:

```
Write-through: keep cache warm with new value after write
Cache invalidation: remove entry, let it be refilled lazily
Write-behind: keep cache warm, defer DB write
```

**For expensive-to-compute or frequently-read data**: prefer write-through (cache stays warm).  
**For infrequently-read data**: prefer invalidation (don't waste cache space).

---

## Key Trade-offs

| Factor | Write-Through | Write-Back |
|---|---|---|
| When DB goes down | Writes fail | Writes succeed (until queue full) |
| When cache crashes | No data loss | Potential data loss (pending writes) |
| Write throughput ceiling | DB max write rate | Higher than DB write rate (batched) |
| Implementation simplicity | Simple | Complex (queue, flush, failure recovery) |

---

## When to use

**Write-through:**
- User-facing data where read-after-write consistency is expected (user updates their own profile and sees it immediately).
- Data that is expensive to fetch from DB and will be read right after being written.
- Moderate write volumes where the 2× write overhead is acceptable.

**Write-back:**
- High-volume write streams: IoT sensor data, analytics events, real-time counters.
- When DB write throughput is the bottleneck and batching would reduce I/O.
- When the application can tolerate potential data loss (e.g., losing ~1 second of metric data is acceptable; losing a financial transaction is not).
- System metrics collection, gaming score updates, activity event logging.

---

## Common Pitfalls

- **Write-back for critical financial data**: never use write-back for data where every write must be durable (payments, orders). Use write-through or skip caching entirely for those writes.
- **Write-through with expensive DB writes**: if DB writes are slow (high write load, complex transactions), write-through doubles the pressure on an already-strained DB.
- **Write-back without a durable queue**: if writes are only in the in-process cache (no Kafka/SQS), a process crash loses all pending writes. Use a durable queue as the intermediary.
- **Cache + DB write not atomic in write-through**: if the DB transaction commits but the cache write fails (timeout, OOM), the cache may temporarily serve the old value. Always handle cache write failure by deleting the key (forces DB fallback on next read).

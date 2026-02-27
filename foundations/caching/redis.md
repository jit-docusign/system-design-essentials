# Redis

## What it is

**Redis** (Remote Dictionary Server) is an **in-memory data structure store** commonly used as a cache, message broker, session store, rate limiter, distributed lock, leaderboard, and pub/sub system. It offers sub-millisecond read/write latency by keeping all data in RAM, optionally backed by disk persistence.

Redis is one of the most ubiquitous infrastructure components in modern web systems. Nearly every large-scale backend uses Redis in some capacity.

---

## How it works

### Architecture

Redis is a **single-threaded** event loop server (I/O multiplexing via epoll). All commands execute atomically in a single thread — no locking required for individual operations. Throughput is typically **100K–1M operations/second** on a single node.

```
Client 1  ─┐
Client 2  ─┤── event loop ──► command execution ──► response
Client N  ─┘    (single thread, sequential)
```

Redis 6+ introduced **I/O threads** for reading/writing network data in parallel while keeping command execution single-threaded.

### Core Data Structures

Redis is not just a key-value store — it provides rich in-memory data structures:

#### String
The most basic type. Stores text, numbers, or binary data (up to 512MB).
```
SET counter 0
INCR counter       → 1  (atomic increment)
INCRBY counter 5   → 6
SET token:abc "user:42" EX 3600   (set with 60-minute TTL)
GET token:abc      → "user:42"
```

#### Hash
A dictionary of field-value pairs within a single key. Efficient for structured objects.
```
HSET user:1001 name "Alice" email "alice@example.com" tier "premium"
HGET user:1001 name        → "Alice"
HGETALL user:1001          → {name: Alice, email: ..., tier: ...}
HINCRBY user:1001 login_count 1
```

#### List
An ordered, doubly-linked list. O(1) push/pop at both ends.
```
RPUSH queue:jobs "job-1" "job-2" "job-3"   (append right)
LPOP queue:jobs           → "job-1"          (pop left = FIFO queue)
BRPOPLPUSH src dst 0      (blocking pop + atomic move = reliable queue)
LRANGE history:user:1 0 9  (last 10 items)
```

#### Set
Unordered collection of unique strings.
```
SADD online:users "user:1" "user:2" "user:3"
SISMEMBER online:users "user:1"  → 1 (true)
SMEMBERS online:users            → ["user:1", "user:2", "user:3"]
SCARD online:users               → 3
SUNION set1 set2                 → union of two sets
SINTER set1 set2                 → intersection
```

#### Sorted Set (ZSet)
Set where each member has a floating-point **score**. Members are stored in ascending score order with O(log n) operations.
```
ZADD leaderboard 9500 "player:alice"
ZADD leaderboard 8200 "player:bob"
ZADD leaderboard 9800 "player:charlie"

ZREVRANGE leaderboard 0 9 WITHSCORES   → top 10 players by score
ZRANK leaderboard "player:alice"        → 1 (zero-indexed rank)
ZINCRBY leaderboard 300 "player:alice"  → 9800 (new score)

ZRANGEBYSCORE leaderboard 9000 +inf    → players with score ≥ 9000
```

Use cases: leaderboards, rate limiting sliding windows, job priority queues, time-indexed event logs.

#### Bitmap
Compact array of bits. Each bit addressed by offset. Efficient for tracking boolean states per user ID.
```
SETBIT user:logins:2024-01-15 user_id 1   (user logged in today)
GETBIT user:logins:2024-01-15 user_id     → 1
BITCOUNT user:logins:2024-01-15           → unique logins today
BITOP AND dest key1 key2                  → users active on both days
```

#### HyperLogLog
Approximate unique count using fixed ~12KB memory regardless of cardinality. Error rate ~0.81%.
```
PFADD page:visits:home "user:1" "user:2" "user:1" "user:3"
PFCOUNT page:visits:home   → ~3 (approximate)
```

#### Stream
Append-only log of messages with consumer groups.
```
XADD events * type "page_view" url "/home" user_id "42"
XREAD COUNT 10 STREAMS events 0      (read last 10 events)
XGROUP CREATE events group1 $        (create consumer group)
XREADGROUP GROUP group1 consumer1 COUNT 5 STREAMS events >
```

### Persistence Options

Redis is in-memory but supports optional persistence:

| Mode | Mechanism | Durability | Performance |
|---|---|---|---|
| **No persistence** | Data lost on restart | None | Fastest |
| **RDB (snapshots)** | Periodic full snapshot to disk | Up to minutes of data loss | Low overhead |
| **AOF (Append-Only File)** | Every write command logged to disk | 1 second (configurable) | Moderate overhead |
| **RDB + AOF** | Both enabled | Seconds | Combined overhead |

For caching use cases: no persistence or RDB is fine (cache can be rebuilt).  
For session store or primary data: AOF with `appendfsync everysec`.

### Redis as a Distributed Cache: Replication

```
Redis Primary  ──WAL stream──►  Replica 1
               ──WAL stream──►  Replica 2
```

Replicas are asynchronous by default. On primary failure, Sentinel or Cluster promotes a replica.

**Redis Sentinel**: monitors Redis instances, performs automatic failover, notifies clients.

**Redis Cluster**: hash-slot-based sharding across multiple primary nodes, each with replicas. 16,384 total slots distributed across nodes.

```
Cluster topology (3 primaries, 1 replica each):
Primary-1: slots 0–5460       + Replica-1
Primary-2: slots 5461–10922   + Replica-2
Primary-3: slots 10923–16383  + Replica-3
```

Clients use cluster-aware Redis clients that route commands to the correct node.

### Transaction Support (MULTI/EXEC)

Redis supports optimistic transactions via MULTI/EXEC:
```
MULTI
  SET key1 "value1"
  INCR counter
  ZADD leaderboard 100 "player-1"
EXEC    ← all commands execute atomically or all fail

DISCARD ← cancel the queued transaction
```

MULTI/EXEC is not true transactions — a command syntax error causes failure, but runtime errors (wrong type) only fail that command, not the whole batch.

For optimistic concurrency: use **WATCH key** before MULTI. If the watched key is modified before EXEC, the transaction aborts.

### Pub/Sub

```
SUBSCRIBE channel:notifications
PUBLISH  channel:notifications "User 42 followed you"
```

Redis pub/sub is fire-and-forget — no persistence, no acknowledgment. Messages are lost if no subscriber is listening. For durable messaging, use Redis Streams.

---

## Key Trade-offs

| Advantage | Disadvantage |
|---|---|
| Sub-millisecond latency | All data must fit in RAM (expensive per GB) |
| Rich data structures beyond simple KV | Async replication = potential data loss on crash |
| Single-threaded = no concurrency bugs per command | Not a primary relational database |
| Atomic operations (INCR, SINTERSTORE, etc.) | Cluster cross-slot transactions not supported |
| Versatile: cache, queue, pub/sub, lock | Cluster client complexity |

---

## When to use

| Use case | Redis feature |
|---|---|
| Application cache | String GET/SET with TTL |
| Session store | String or Hash with TTL |
| Rate limiting | INCR + EXPIRE or Sorted Sets |
| Leaderboard | Sorted Set (ZADD/ZREVRANGE) |
| Task queue | List (RPUSH/BLPOP) or Stream |
| Pub/Sub messaging | Pub/Sub or Streams |
| Distributed lock | SET NX EX + Lua script |
| Feature flags | String or Hash |
| Unique visitor count | HyperLogLog |
| User activity bitmap | Bitmap |

---

## Common Pitfalls

- **No persistence for critical data**: using Redis as the only store for non-cache data without persistence → data lost on restart. Add AOF for durability.
- **Memory overflow without eviction policy**: without a configured `maxmemory-policy`, Redis will reject writes when memory is full. Set an appropriate eviction policy and `maxmemory` limit.
- **Hot key bottleneck**: a single key receiving millions of requests per second hits a single Redis thread and a single node. Spread hot keys using local in-process caching or key sharding (`key:shard:{hash%N}`).
- **Large value storage**: storing megabyte-size values in Redis saturates network and serialization overhead. Keep values small; store large binary data in object storage (S3) and cache only the metadata/URL in Redis.
- **Not using pipelining for batch operations**: sending 1000 individual GET commands has 1000 round-trip latencies. Use `MGET` or pipelining to batch them into one round-trip.

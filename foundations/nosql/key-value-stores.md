# Key-Value Stores

## What it is

A **key-value store** is the simplest NoSQL data model: data is stored as a collection of **key → value** pairs. Every read and write is addressed by an opaque key; the database has no knowledge of the value's structure.

Key-value stores prioritize extreme **read/write speed** and **linear horizontal scalability** over rich query capabilities. They are the foundation of caching layers and session stores at every scale of web infrastructure.

**Common systems**: Redis, Memcached, DynamoDB (simple key access), etcd, Apache Ignite, Aerospike.

---

## How it works

### Data Model

```
SET user:session:abc123       →   {"user_id": 42, "expires": 1720000000}
GET user:session:abc123       ←   {"user_id": 42, "expires": 1720000000}

SET product:price:sku-9001    →   "29.99"
GET product:price:sku-9001    ←   "29.99"

DEL user:session:abc123       →   (key deleted)
```

Keys are typically strings. Values are opaque bytes — the store doesn't interpret or query inside them. Applications serialize values (JSON, MessagePack, Protobuf) before storing.

### Operations

Most key-value stores offer a minimal operation set:
- `GET key` → value (or null if missing)
- `SET key value` → store
- `DEL key` → delete
- `EXISTS key` → boolean
- `EXPIRE key ttl` → set time-to-live (auto-expiry)
- Atomic increment: `INCR key`, `INCRBY key delta`

Richer stores like Redis add data structure support beyond opaque blobs:

| Redis Data Type | Use case |
|---|---|
| String | Counters, cached values, session tokens |
| Hash | User profiles (field-level access), config |
| List | Message queues, activity logs, leaderboards |
| Set | Unique tag/category memberships, deduplication |
| Sorted Set | Leaderboards, priority queues, range queries by score |
| Bitmap | Compact boolean flags, user activity tracking |
| HyperLogLog | Approximate unique count (very memory-efficient) |
| Stream | Append-only log, event sourcing, pub/sub |

### Common Use Cases in Detail

#### Session Storage
HTTP is stateless. Session data (user identity, cart contents) must be stored server-side and looked up by a session token on every request:
```
SET session:{token}  {json-blob}  EX 3600   -- 1-hour TTL
GET session:{token}
```
Every request does one O(1) GET. Session auto-expires when the user is idle.

#### Caching
The canonical use case. Expensive database queries or API responses are cached:
```
GET cache:user:{id}
  → cache miss → fetch from DB → SET cache:user:{id} {value} EX 300
  → cache hit → return value immediately
```
See [Caching](../caching/) for a comprehensive treatment.

#### Rate Limiting
Atomic increment + expiry enables simple rate limiting:
```
INCR rate:ip:{ip_address}
EXPIRE rate:ip:{ip_address} 60   -- reset after 1 minute

if GET rate:ip:{ip_address} > 100:
    reject request (429 Too Many Requests)
```
See [Rate Limiting](../traffic/rate-limiting.md).

#### Distributed Locks
`SET key value NX EX 30` (Set if Not eXists, expire in 30s) is the foundation of a distributed lock:
```
SET lock:resource:order-456  {lock-id}  NX EX 30
  → OK if acquired, nil if already held
```
See [Distributed Locks](../distributed/distributed-locks.md).

#### Feature Flags / Config
Application configuration quickly retrievable at startup or per-request:
```
GET feature:dark-mode-enabled  → "true"
GET config:max-upload-bytes    → "10485760"
```

### Distribution and Clustering

**Memcached**: purely in-memory, no persistence, no replication. Clients do client-side consistent hashing across pool members. If a node fails, its data is simply gone (acceptable for caches).

**Redis Cluster**: automatic hash-slot-based sharding (16,384 slots distributed across nodes). Each shard has a primary + replicas. If a primary fails, a replica is promoted automatically. Clients are redirected to the correct node by the cluster protocol.

**DynamoDB**: fully managed key-value store with automatic sharding. Table partition key maps to a node via consistent hashing. Provides configurable consistency (eventually consistent reads vs strongly consistent reads).

### Memory and Eviction

Key-value stores are typically memory-first. When memory is full, a configured eviction policy removes old keys:

| Policy | Behavior |
|---|---|
| `noeviction` | Reject writes when memory is full |
| `allkeys-lru` | Evict least-recently-used key from all keys |
| `volatile-lru` | Evict LRU key only from keys with TTL set |
| `allkeys-lfu` | Evict least-frequently-used key |
| `volatile-ttl` | Evict key with nearest TTL expiry |

Redis supports optional persistence via RDB snapshots and AOF (Append-Only File) so in-memory data can be restored after restart.

---

## Key Trade-offs

| Advantage | Disadvantage |
|---|---|
| O(1) GET/SET operations | Cannot query by value — must know the key |
| Extreme throughput (millions ops/sec) | No rich query language |
| Simple horizontal scaling | Very limited data relationships |
| TTL-based auto-expiry | Consistency guarantees vary by system |
| Low latency (sub-millisecond) | Memory-bound storage (expensive per GB) |

---

## When to use

| Workload | Appropriate choice |
|---|---|
| Session store | Redis or Memcached |
| Application cache | Redis or Memcached |
| Rate limiting counters | Redis (INCR + EXPIRE) |
| Leaderboards / sorted ranking | Redis Sorted Sets |
| Pub/sub message bus | Redis Pub/Sub or Streams |
| Feature flags / config | Redis or etcd |
| Distributed lock | Redis (RedLock) or etcd |
| Simple metadata lookup | DynamoDB (key-value mode) |

---

## Common Pitfalls

- **No backup or persistence configured**: using Redis as a primary data store without AOF or RDB configured means data loss on restart. Choose persistence mode based on durability requirements.
- **Hot key problem**: if millions of requests hit the same key (e.g., a global counter or a trending topic), that key becomes a bottleneck on a single node. Spread hot keys with key-sharding or local caching.
- **Unbounded key growth**: keys without TTL accumulate indefinitely, exhausting memory over time. Always set TTLs on session and cache keys.
- **Cache stampede**: when a popular cached value expires, many concurrent requests all miss cache simultaneously and flood the origin database. Use short-circuit locking ("dog-pile locking") or probabilistic early expiration to prevent this.
- **Using key-value for relational data**: if your access patterns require querying by multiple attributes or joining, a key-value store is the wrong tool. Add a relational or document database for that data.

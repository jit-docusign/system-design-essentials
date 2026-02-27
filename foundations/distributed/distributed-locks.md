# Distributed Locks

## What it is

A **distributed lock** (also called a distributed mutex) is a mechanism that ensures **only one process or node in a distributed system can execute a critical section at a time** — preventing race conditions and duplicate work across multiple servers.

Unlike a thread lock in a single process (which uses in-memory primitives), a distributed lock uses a **shared external system** (Redis, ZooKeeper, PostgreSQL) as the coordination medium, since multiple independent processes have no shared memory.

---

## How it works

### What Requires a Distributed Lock

Race condition examples in distributed systems:

```
Without a distributed lock:
  Worker 1 and Worker 2 both poll a job queue
  Both see job-001 in the queue
  Both start processing job-001
  Result: job processed twice → duplicate order, double charge, duplicate email

  Session cleanup job runs at midnight
  Two instances spin up simultaneously (scheduler bug)
  Both start cleaning → concurrent writes corrupt session data
```

### Lock Properties

A correct distributed lock must provide:
- **Mutual exclusion**: only one client holds the lock at any time.
- **Deadlock safety**: locks must eventually be released, even if the holder crashes (TTL/lease).
- **Fault tolerance**: the lock must survive individual node failures.

### Redis-Based Locking (SET NX EX)

The simplest Redis-based lock uses `SET ... NX EX`:

```python
import redis
import uuid

r = redis.Redis()

def acquire_lock(lock_name: str, ttl_seconds: int = 30) -> str | None:
    """Acquire a lock. Returns a token if acquired, None if not."""
    token = str(uuid.uuid4())
    acquired = r.set(f"lock:{lock_name}", token, nx=True, ex=ttl_seconds)
    return token if acquired else None

def release_lock(lock_name: str, token: str) -> bool:
    """Release lock only if we own it (compare-and-delete)."""
    # Lua script: atomic check + delete
    lua_script = """
    if redis.call("GET", KEYS[1]) == ARGV[1] then
        return redis.call("DEL", KEYS[1])
    else
        return 0
    end
    """
    result = r.eval(lua_script, 1, f"lock:{lock_name}", token)
    return bool(result)

# Usage
token = acquire_lock("order-processor", ttl_seconds=10)
if token:
    try:
        process_order()
    finally:
        release_lock("order-processor", token)
else:
    print("Could not acquire lock — another instance is working")
```

**Why compare-and-delete?** Without it:
```
Worker 1 acquires lock with token ABC
Worker 1 takes too long → TTL expires → lock released
Worker 2 acquires lock with token XYZ
Worker 1 finishes → releases lock by key (without checking token) → deletes Worker 2's lock
Worker 3 immediately acquires the lock that Worker 2 still thinks it holds
→ Two workers both think they hold the lock
```

Always use a unique token and compare-and-delete (via Lua script for atomicity).

### Redlock Algorithm (Multi-Node Redis)

Single-node Redis is a SPOF for locks. **Redlock** uses N ≥ 3 independent Redis nodes:

```
1. Record current time.
2. Try to acquire the lock on N nodes sequentially with a small timeout each.
3. Success: lock acquired on majority (N/2 + 1) within total elapsed time < TTL.
4. Effective TTL = initial TTL - elapsed_time.
5. Failure: release on all nodes and retry with backoff.
```

```python
# Example with redlock-py library
from redlock import Redlock, MultipleRedlockException

dlm = Redlock([
    {"host": "redis1", "port": 6379},
    {"host": "redis2", "port": 6379},
    {"host": "redis3", "port": 6379}
])

try:
    lock = dlm.lock("order-processor", 10000)  # TTL = 10s in ms
    if lock:
        try:
            process_order()
        finally:
            dlm.unlock(lock)
    else:
        handle_lock_unavailable()
except MultipleRedlockException:
    handle_lock_unavailable()
```

Note: Redlock is controversial (see Martin Kleppmann's critique and Antirez's response). The debate is about clock skew and whether "locking with TTL" is sufficient for safety without fencing tokens.

### Fencing Tokens

A **fencing token** solves the problem of a lock holder that becomes preempted (GC pause, slow network) and takes an action after its TTL has expired:

```
Client 1 acquires lock → gets token 1
Client 1: long GC pause
  Lock TTL expires
  Client 2 acquires lock → gets token 2
  Client 2 writes to storage with token 2
  Client 1 resumes → tries to write with token 1
    Storage rejects: "token 1 is older than 2, rejecting"
```

The storage layer enforces monotonically increasing token ordering, preventing stale writes even after lock expiry.

### ZooKeeper-Based Distributed Lock

ZooKeeper provides sequential ephemeral znodes as a coordination primitive for fair, queued locks:

```
1. Client creates an ephemeral sequential znode: /locks/order-proc-0000000042
2. Client lists all children of /locks/: [0000000040, 0000000041, 0000000042]
3. If client's znode is the smallest → acquired the lock
4. If not → watch the next-smaller znode (0000000041)
5. When 0000000041 is deleted → re-check → if now smallest → acquired

On client crash: ephemeral znode automatically deleted → next waiter acquires lock
```

ZooKeeper locks are fair (FIFO) and fault-tolerant — client crash automatically releases the lock.

---

## Key Trade-offs

| Advantage | Disadvantage |
|---|---|
| Prevents race conditions and duplicate processing | Network partition can cause lock loss or split-brain |
| TTL ensures locks are released on crash | Under GC pauses, clock skew, a process can act after its lock expires |
| Simple Redis implementation widely available | Adds latency — lock acquisition is a network round-trip |
| ZooKeeper provides stronger guarantees | Higher complexity at the system level |

---

## When to use

Distributed locks are appropriate for:
- **Cron/scheduled jobs** that must run on exactly one instance.
- **Resource deduplication**: ensuring a unique constraint across services that own separate databases.
- **Rate limiting by resource** (e.g., one update per user at a time).
- **Coordinating writes to shared external systems** (calling a payment gateway that doesn't support idempotency).

Prefer **idempotency + at-least-once processing** over distributed locks whenever possible — locks add latency and are a coordination bottleneck. Use locks only when idempotency alone is insufficient.

---

## Common Pitfalls

- **Not setting a TTL**: a lock without TTL will be held forever if the holder crashes. Always set a generous but bounded TTL.
- **Not using compare-and-delete**: releasing without verifying token ownership can release another holder's lock. Always use the Lua script check.
- **TTL too short**: if the processing time occasionally exceeds the TTL (GC pause, slow query), the lock expires while work is in progress. Use a watchdog thread to extend the TTL, or use a heartbeat lease model (Redisson's WatchdogTimeout pattern).
- **Using a distributed lock to paper over design problems**: if you need a lock to prevent duplicate processing, the underlying design may be flawed. First consider idempotency, deduplication tables, and CRDT-based approaches before reaching for a lock.

# Design a Distributed Job Scheduler

## 1. What It Is / Problem Statement

A distributed job scheduler executes tasks (jobs) at a specified time or on a recurring schedule, across a fleet of worker nodes, with strong guarantees around **exactly-once execution**, **fault tolerance**, and **observability**.

### Scale targets (Cron-as-a-Service at hyperscale)
| Dimension | Target |
|---|---|
| Registered jobs | 100 million |
| Jobs triggered per second | 10,000 |
| Minimum scheduling granularity | 1 second |
| Max job execution time | Hours (long-running) |
| Availability | 99.99% (< 53 min/year downtime) |
| Scheduling accuracy | ≤ 1 second late |

### Why it's hard
- **Clock skew** — no single global clock; nodes disagree on "now"
- **Exactly-once guarantee** — a job must not be skipped or double-fired even across failovers
- **Scale** — scanning 100M jobs every second is computationally infeasible without careful data modeling
- **Long-running jobs** — a job that runs for hours must survive worker crashes and be resumable
- **Hot second problem** — many crons are set to `:00` of the hour, causing thundering herds

---

## 2. Architecture Overview

```
                          ┌─────────────────────────────────────────────┐
                          │              Clients / APIs                  │
                          │  (register job, pause, trigger, get status) │
                          └─────────────────┬───────────────────────────┘
                                            │ REST / gRPC
                          ┌─────────────────▼───────────────────────────┐
                          │              API Service                     │
                          │  - CRUD for job definitions                  │
                          │  - Auth, validation, idempotency             │
                          └────────┬──────────────────┬─────────────────┘
                                   │                  │
                     ┌─────────────▼──────┐  ┌────────▼──────────────────┐
                     │   Job Store (DB)   │  │  Schedule Index (DB/Redis) │
                     │  job definitions   │  │  next_run_at → job_id      │
                     │  (MySQL/Postgres)  │  │  sorted by next_run_at     │
                     └────────────────────┘  └────────┬──────────────────┘
                                                       │ poll every 1s
                          ┌────────────────────────────▼────────────────┐
                          │            Scheduler (Leader)                │
                          │  - Scans index window [now, now+5s]         │
                          │  - Enqueues due jobs                         │
                          │  - Updates next_run_at (cron advance)        │
                          └────────────────────────────┬────────────────┘
                                                        │ push
                          ┌─────────────────────────────▼───────────────┐
                          │           Job Queue (Kafka / Redis)          │
                          │  Partitioned by job_type or tenant           │
                          └───────────┬─────────────────────────────────┘
                           ┌──────────┤──────────┐
             ┌─────────────▼──┐ ┌─────▼──────┐ ┌─▼──────────────┐
             │   Worker A     │ │  Worker B  │ │   Worker C     │
             │  (executor)    │ │ (executor) │ │  (executor)   │
             └─────┬──────────┘ └─────┬──────┘ └───────┬────────┘
                   │                  │                 │
             ┌─────▼──────────────────▼─────────────────▼────────┐
             │              Execution Store (DB)                   │
             │  job_runs: run_id, job_id, status, worker, retries  │
             └─────────────────────────────────────────────────────┘
                          │
             ┌────────────▼──────────────────┐
             │     Observability / UI         │
             │  logs, metrics, audit trail    │
             └───────────────────────────────┘
```

---

## 3. How It Works

### 3.1 Job Definition Model

```sql
CREATE TABLE jobs (
    job_id        UUID PRIMARY KEY,
    tenant_id     UUID NOT NULL,
    name          VARCHAR(255),
    schedule      VARCHAR(100),       -- cron expression or ISO 8601 interval
    timezone      VARCHAR(64)  DEFAULT 'UTC',
    payload       JSONB,              -- passed to executor as-is
    executor_type VARCHAR(64),        -- http_callback | kafka_topic | lambda | shell
    executor_config JSONB,            -- URL, topic, function ARN, etc.
    max_retries   INT          DEFAULT 3,
    retry_backoff VARCHAR(32)  DEFAULT 'exponential',
    timeout_sec   INT          DEFAULT 300,
    status        VARCHAR(16)  DEFAULT 'active',  -- active | paused | deleted
    next_run_at   TIMESTAMPTZ,
    created_at    TIMESTAMPTZ  DEFAULT now(),
    updated_at    TIMESTAMPTZ  DEFAULT now()
);

-- Execution audit log
CREATE TABLE job_runs (
    run_id        UUID PRIMARY KEY,
    job_id        UUID NOT NULL REFERENCES jobs(job_id),
    scheduled_at  TIMESTAMPTZ NOT NULL,
    started_at    TIMESTAMPTZ,
    finished_at   TIMESTAMPTZ,
    status        VARCHAR(16),   -- pending | running | success | failed | timed_out
    worker_id     VARCHAR(128),
    attempt       INT DEFAULT 1,
    output        JSONB,
    error_msg     TEXT
);
```

---

### 3.2 Scheduling: Efficient Polling with a Time-Indexed Queue

Scanning 100M rows every second is too slow. Instead, maintain a **sorted index** of `next_run_at`:

```sql
CREATE INDEX idx_jobs_next_run ON jobs (next_run_at)
    WHERE status = 'active';
```

The scheduler queries a small window ahead of now:

```sql
-- Run every 1 second in the scheduler loop
SELECT job_id, schedule, timezone, payload, executor_config
FROM   jobs
WHERE  status      = 'active'
  AND  next_run_at >= now()
  AND  next_run_at <  now() + INTERVAL '5 seconds'
FOR UPDATE SKIP LOCKED;   -- prevents double-fetch in multi-scheduler setup
```

`FOR UPDATE SKIP LOCKED` is the key: multiple scheduler replicas can run safely without a separate distributed lock round-trip.

After fetching, calculate `next_run_at` using the cron expression:

```python
import croniter
from datetime import datetime, timezone

def advance_next_run(schedule: str, tz: str, from_time: datetime) -> datetime:
    cron = croniter.croniter(schedule, from_time)
    return cron.get_next(datetime).astimezone(timezone.utc)
```

Then update in the same transaction:

```python
cursor.execute("""
    UPDATE jobs
    SET    next_run_at = %s,
           updated_at  = now()
    WHERE  job_id = %s
""", (next_run, job_id))
```

---

### 3.3 Exactly-Once Execution via Idempotency Keys

A job must not execute twice for the same scheduled slot, even if the scheduler crashes after enqueueing but before committing.

**Solution: Transactional outbox pattern**

```
┌────────────────────────────────────────────────────────┐
│   DB Transaction                                        │
│   1. INSERT job_runs (run_id, job_id, scheduled_at,    │
│                       status='pending')                 │
│   2. UPDATE jobs SET next_run_at = <next>              │
│   3. INSERT outbox (run_id, payload) ← message staging │
└────────────────────────────────────────────────────────┘
         │
         │  Outbox relay (Debezium CDC or polling relay)
         ▼
    Kafka / queue  →  Workers

Worker ACKs run_id → outbox row deleted
```

**Idempotency on workers**: Workers receive `run_id` in the message and check:

```python
def execute_job(run_id, job_id, payload):
    # Idempotent guard
    if db.exists("SELECT 1 FROM job_runs WHERE run_id=%s AND status='running'", run_id):
        return  # already being processed
    db.execute("UPDATE job_runs SET status='running', worker_id=%s, started_at=now()
                WHERE run_id=%s AND status='pending'", (worker_id, run_id))
    if rows_affected == 0:
        return  # another worker grabbed it (optimistic lock)
    # ... do work ...
```

---

### 3.4 Scheduler High Availability: Leader Election

Only **one scheduler** should scan and enqueue at a time to avoid double-firing. Run multiple scheduler replicas with leader election using a distributed lock:

```
┌─────────────────────────────────────────────────────┐
│  scheduler-1 (leader)        scheduler-2 (standby)  │
│  Holds Redis lock             Tries to acquire lock  │
│  "scheduler:leader" TTL=10s                          │
│  Renews every 5s               Monitors lock         │
└─────────────────────────────────────────────────────┘
```

```python
import redis

r = redis.Redis()
LOCK_KEY   = "scheduler:leader"
LOCK_TTL   = 10  # seconds
MY_ID      = socket.gethostname()

def try_become_leader() -> bool:
    return r.set(LOCK_KEY, MY_ID, nx=True, ex=LOCK_TTL)

def renew_lease() -> bool:
    # Use Lua to prevent renewing someone else's lock
    script = """
    if redis.call("get", KEYS[1]) == ARGV[1] then
        return redis.call("expire", KEYS[1], ARGV[2])
    else
        return 0
    end
    """
    return bool(r.eval(script, 1, LOCK_KEY, MY_ID, LOCK_TTL))
```

If the leader dies (misses renewal), the standby wins the next `SET NX` and takes over within TTL seconds.

---

### 3.5 Worker Execution and Retry Logic

```
┌────────────────────────────────────────────────────────────────┐
│                       Worker Lifecycle                          │
│                                                                 │
│  Dequeue message → Mark run as 'running'                       │
│         │                                                       │
│         ▼                                                       │
│  Execute job (HTTP callback / Kafka produce / Lambda invoke)   │
│         │                                                       │
│    ┌────┴─────┐                                                 │
│  success   failure / timeout                                    │
│    │             │                                              │
│  Mark       attempt < max_retries?                             │
│  'success'    ├── yes → re-enqueue with delay (backoff)        │
│               └── no  → Mark 'failed', alert                   │
└────────────────────────────────────────────────────────────────┘
```

**Exponential backoff with jitter**:

```python
import random

def backoff_delay(attempt: int, base: float = 1.0, cap: float = 300.0) -> float:
    delay = min(cap, base * (2 ** attempt))
    jitter = random.uniform(0, delay * 0.3)
    return delay + jitter
```

**Heartbeat for long-running jobs**:

Workers running multi-minute jobs must periodically extend their lease:

```python
# Worker sends heartbeat every 30s
db.execute("""
    UPDATE job_runs
    SET    heartbeat_at = now()
    WHERE  run_id = %s AND status = 'running'
""", (run_id,))
```

A reaper process marks jobs `timed_out` if `heartbeat_at < now() - timeout_sec`:

```python
db.execute("""
    UPDATE job_runs
    SET    status = 'timed_out', finished_at = now()
    WHERE  status = 'running'
      AND  heartbeat_at < now() - make_interval(secs => timeout_sec)
""")
```

---

### 3.6 Handling the "Hot Second" Problem

Many cron jobs are scheduled at `:00` of every hour (e.g., `0 * * * *`). At 14:00:00 UTC, millions of jobs become due simultaneously.

**Mitigation strategies**:

| Strategy | Mechanism |
|---|---|
| Scheduled hash-based skew | Add `hash(job_id) % 60` seconds to initial `next_run_at` for non-exact jobs |
| Multi-partition queue | Kafka partitioned by `tenant_id`; avoids single-partition bottleneck |
| Token bucket per tenant | Limit each tenant to N dispatches/sec to prevent noisy-neighbor spikes |
| Tiered scan window | Scan ahead by 5–60s; jobs are ready before the second arrives |
| Priority lanes | Separate queues for `critical`, `standard`, `batch` job tiers |

---

### 3.7 Distributed Architecture: Partitioned Schedulers

At extreme scale (10K jobs/sec), a single scheduler becomes a bottleneck. Partition jobs across multiple schedulers:

```
                   Job → hash(job_id) % N → Scheduler shard N
                         
┌────────────────┐   ┌────────────────┐   ┌────────────────┐
│ Scheduler-0    │   │ Scheduler-1    │   │ Scheduler-2    │
│ owns jobs      │   │ owns jobs      │   │ owns jobs      │
│ where          │   │ where          │   │ where          │
│ job_id % 3 = 0 │   │ job_id % 3 = 1 │   │ job_id % 3 = 2 │
└────────────────┘   └────────────────┘   └────────────────┘
```

Each shard independently polls its subset of the index. Adding a shard requires rehashing assignments — use consistent hashing to minimize migration.

---

## 4. Key Design Decisions

### Decision 1: Polling vs. Priority Queue for Scheduling

| Approach | How It Works | Pros | Cons |
|---|---|---|---|
| **DB polling with time index** ✓ | Sorted index on `next_run_at`; scheduler polls every 1s | Simple, durable, ACID; works for millions of jobs | DB is the bottleneck at extreme rates |
| Redis Sorted Set | `ZADD due_jobs <timestamp> <job_id>` → `ZRANGEBYSCORE` every second | Sub-millisecond scan; O(log N + k) | Redis is not the source of truth; data can lag on crash |
| Hierarchical timing wheel | Slots at 1s resolution; buckets hold due job IDs | O(1) schedule and fire; memory-efficient | Complex; doesn't persist natively; must back with DB |
| Time-partitioned Kafka topics | Jobs in future partitions; consumers run at right time | Naturally durable | Low per-second resolution without complex offsets |

**Chosen: DB polling + Redis cache for hot-path reads.** Simple, auditable, works up to ~10K/sec with a secondary sorted index.

---

### Decision 2: Exactly-Once Delivery

| Strategy | Guarantee | Trade-off |
|---|---|---|
| **Transactional outbox** ✓ | Near exactly-once | Requires CDC/relay process; slight delay |
| Distributed lock before enqueue | Prevents double-enqueue | Lock failure window exists; adds latency |
| Deduplication on worker (run_id check) | Idempotent execution | At-least-once delivery; job logic must be idempotent |
| Kafka transactions (EOS) | Exactly-once in queue | Kafka-specific; doesn't cover worker crashes |

**Chosen: Outbox + worker-side idempotency (defense in depth).**

---

### Decision 3: Executor Delivery Model

| Model | Use Case | Pros | Cons |
|---|---|---|---|
| **HTTP callback** ✓ | Most general; caller owns logic | Decoupled; any language | Caller must be reachable; retry storms possible |
| Kafka topic publish | Async pipelines | High throughput; durable | Consumer must be online to process |
| Lambda / FaaS invocation | Serverless workloads | Scales to zero; cheap | Cold start; function timeout limits |
| SSH/shell exec | Legacy/internal scripts | Simple | Security risk; hard to scale |

**Chosen: HTTP callback as the primary model**, with Kafka publish and Lambda as optional executor types per job config.

---

### Decision 4: Long-Running Job Coordination

| Approach | Description | Trade-off |
|---|---|---|
| **Heartbeat + reaper** ✓ | Worker pings DB; reaper times out stale runs | Simple; graceful failure detection |
| Worker lease renewal (Redis) | Worker renews a Redis TTL key | Fast detection (< TTL); Redis dependency |
| Callback on completion only | Worker signals done when finished | Simple but gives no visibility into hung jobs |

---

## Hard-Learned Engineering Lessons

1. **Polling the entire jobs table every second** — forgetting to index `next_run_at` leads to full-table scans. Always use a filtered, sorted index and only read a narrow time window.

2. **Ignoring timezone handling** — cron expressions like `0 9 * * *` mean 9 AM in the job's configured timezone, not UTC. Forgetting DST transitions causes jobs to fire at the wrong time twice a year.

3. **Omitting the exactly-once discussion** — most teams say "use a queue" without explaining how they prevent the scheduler from re-enqueueing on crash recovery. Transactional outbox or `FOR UPDATE SKIP LOCKED` is the correct answer, and knowing why matters at any scale.

4. **Single scheduler = SPOF** — the system needs a leader-election mechanism (Redis lock, Zookeeper ephemeral node, or DB advisory lock) so standby schedulers take over within seconds.

5. **No heartbeat for long-running jobs** — claiming a job is "running" without heartbeating means it will appear stuck forever after the worker crashes. A reaper process is necessary.

6. **Ignoring the hot-second thundering herd** — `0 * * * *` fires for millions of jobs simultaneously. Without queue partitioning and per-tenant rate limiting, the entire queue layer spikes every hour.

7. **Confusing cron granularity** — standard cron has 1-minute granularity. For sub-minute scheduling (every 5 seconds), you need a different mechanism: a polling loop with second-resolution timestamps, or a timer wheel — not a cron expression.

8. **Not surfacing execution history** — treating the scheduler as fire-and-forget with no audit log makes debugging production incidents impossible. `job_runs` with full status history is non-negotiable.

9. **Cascading retry storms** — without exponential backoff with jitter, a failing job retrying every second can overwhelm the execution target. Jitter breaks synchronized retry waves across concurrent failing jobs.

10. **Static worker pool sizing** — not accounting for bursty job patterns (end of month batch jobs, hourly rollups). Workers should auto-scale based on queue depth metrics.

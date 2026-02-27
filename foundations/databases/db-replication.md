# Database Replication

## What it is

**Database replication** is the process of copying data from one database node (the **primary**) to one or more additional nodes (**replicas / secondaries**) in near-real-time. Replication achieves two goals:
1. **High availability**: if the primary fails, a replica can be promoted so the system continues accepting writes.
2. **Read scaling**: read queries are distributed across replicas, reducing load on the primary.

Replication is almost always the first scaling mechanism applied to a relational database.

---

## How it works

### Primary-Replica (Leader-Follower) Model

The most common topology:
- One **primary** accepts all write operations.
- One or more **replicas** receive a stream of changes from the primary and apply them in order.
- Reads can be served by any replica (with consistency caveats).

```
        writes
Client ────────► Primary (read/write)
                    │
              WAL stream
             ┌──────┴──────┐
             ▼             ▼
        Replica-1       Replica-2
      (read-only)     (read-only)
             │
          reads
```

### Replication Mechanisms

#### Statement-Based Replication (SBR)
The primary sends the actual SQL statements to replicas, which re-execute them.
- **Problem**: non-deterministic functions (`NOW()`, `RAND()`, `UUID()`) produce different results on each node → divergence.
- MySQL supports SBR but it's fragile; mostly superseded.

#### Row-Based Replication (RBR)
The primary sends **the actual data changes** (before and after images of rows) rather than SQL statements.
- Deterministic and safe for all operations.
- Higher bandwidth than SBR for bulk updates.
- **PostgreSQL logical replication** and **MySQL binlog row format** use this.

#### WAL Shipping (Physical Replication)
PostgreSQL's default streaming replication sends **WAL (Write-Ahead Log) segments** — the exact byte-level changes to data files.
- **Pros**: exact byte-for-byte copy of primary; very low overhead.
- **Cons**: replica must be same major PostgreSQL version; no partial replication (all tables, all databases on the server).

### Synchronous vs Asynchronous Replication

| Mode | How it works | Pros | Cons |
|---|---|---|---|
| **Async** (default) | Primary commits immediately; replica applies changes eventually | Low write latency | Replica may lag; potential data loss on primary crash |
| **Sync** | Primary waits for replica to confirm write before committing | Zero data loss (if properly configured) | Higher write latency; replica failure blocks primary writes |
| **Semi-sync** | Primary waits for at least one replica to acknowledge | Balance between safety and performance | Slight write overhead |

PostgreSQL's `synchronous_standby_names` controls which replicas must confirm before a commit returns.

### Replication Lag

**Replication lag** is the delay between a write being committed on the primary and being visible on a replica. Under normal conditions, lag is milliseconds. Under heavy write load, it can grow to seconds or minutes.

Problems caused by replication lag:
- **Read-your-own-writes violation**: user writes a post, then immediately re-reads it from a replica that hasn't replicated it yet — user sees a "missing" post.
- **Monotonic reads violation**: user's second read sees older data than their first read because different replicas are queried.
- **Stale data for business logic**: a payment confirmation query might read "unpaid" from a lagging replica and re-charge the user.

**Mitigations:**
- Read recent writes from the primary for a short time after a write.
- Monor reads for a given user session to the same replica.
- Use synchronous replication for critical consistency requirements.
- Check replication lag before routing reads to a replica.

### Read Replica Routing

Application-level routing (most common):
```
Read queries  → replica pool (load balanced)
Write queries → primary exclusively
```

With connection proxies like **ProxySQL**, **pgBouncer + pgCat**, or **AWS RDS Proxy**:
```
App → Proxy → (route by query type) → Primary or Replica pool
```

Many ORM libraries (Django, Rails ActiveRecord) have native read/write splitting support.

### Failover Promotion

When the primary fails:
1. Replica with least lag is selected as the new primary.
2. Replica is promoted: it stops applying WAL and starts accepting writes.
3. Other replicas are reconfigured to follow the new primary.
4. DNS or load balancer routes write traffic to new primary.

Tools that automate this: **Patroni** (PostgreSQL), **Orchestrator** (MySQL), **AWS RDS Multi-AZ** (managed).

**Split-brain risk**: if the old primary comes back online without knowing it was replaced, two nodes may both accept writes. Fencing (STONITH — "Shoot the Other Node in the Head") prevents this by ensuring the old primary is definitively shut down before the new primary is promoted.

### Replication Factor

Beyond a single replica for redundancy, typical deployments use:
- **1 primary + 2 replicas** for high availability (quorum of 3 with one failure tolerated).
- More replicas for high read scalability or cross-region deployments.

Cross-region replication is inherently asynchronous (speed of light delay).

---

## Key Trade-offs

| Feature | Trade-off |
|---|---|
| Async replication | Low write latency vs potential data loss |
| Sync replication | Zero data loss vs higher write latency |
| More replicas | More read throughput vs more replication bandwidth from primary |
| Replica reads | Scale reads vs risk of reading stale data |
| Failover automation | Faster recovery vs split-brain risk if not carefully designed |

---

## When to use

- Your application is **read-heavy** (80–90% reads): replicas distribute load without touching the primary.
- You need **high availability** for the database layer (automatic failover).
- You have **reporting or analytics queries** that are slow and would impact production — run them against a dedicated replica.
- You need a **hot standby** for disaster recovery.
- You run **blue/green deployments** and want a replica to partially warm its cache before promotion.

---

## Common Pitfalls

- **Reading critical data from replicas**: business-critical reads (e.g., inventory check during checkout) must come from the primary. Don't let "route all reads to replicas" become an absolute rule.
- **Ignoring replication lag**: treating replica reads as fully consistent leads to subtle correctness bugs. Measure lag continuously and alert on it.
- **No automated failover**: manually detecting and promoting a replica during an incident takes 10–30 minutes. Use a failover automation tool (Patroni, RDS Multi-AZ).
- **Primary as SPOF for writes**: replication helps reads and HA, but the primary is still a single write bottleneck. If writes grow too large, sharding is needed.
- **Not testing failover**: a replica that has never been promoted is an untested assumption. Regularly drill failovers to ensure they work.

# Read Replicas

## What it is

A **read replica** is a copy of a database that receives a continuous stream of changes from the primary and is used exclusively to serve **read queries**. All writes still go to the primary; read queries are offloaded to one or more replicas.

Read replicas are the most common first step in scaling a relational database beyond a single node. They increase read throughput without the complexity of sharding.

---

## How it works

### Architecture

```
              Writes
Application ──────────► Primary (R/W)
                │             │
            Reads     WAL replication stream
                │        ┌────┴────┐
                └────►  Replica 1  Replica 2
                        (Read)    (Read)

Reads distributed across replicas
Writes always to primary only
```

See [Database Replication](../databases/db-replication.md) for the technical details of how replication works (WAL streaming, row-based, async vs sync).

### What Can Be Safely Routed to a Replica

**Safe to read from replica (can tolerate short staleness):**
- Product catalog listings
- Blog posts, articles, media
- User statistics (follower count, post count)
- Historical reports and analytics
- Search results (if you don't use Elasticsearch)
- Any list that doesn't require the absolute latest write to be visible

**Must read from primary (require latest data):**
- Payment and balance queries immediately after a transaction
- Inventory checks during checkout (race conditions matter)
- Read-your-own-writes scenarios (profile page right after editing)
- Authentication state (login, password reset token validity)
- Any action that depends on a value the same user just wrote

### Routing Strategies

#### Application-Level Routing

The application explicitly chooses primary vs replica based on the operation:

```python
# Python example with SQLAlchemy

class UserRepository:
    def __init__(self):
        self.primary = create_engine(PRIMARY_DB_URL, pool_size=20)
        self.replica  = create_engine(REPLICA_DB_URL, pool_size=20)

    def update_profile(self, user_id, data):
        # Writes always go to primary
        with self.primary.connect() as conn:
            conn.execute("UPDATE users SET ...", data)

    def get_profile(self, user_id):
        # Profile reads can go to replica (slight staleness acceptable)
        with self.replica.connect() as conn:
            return conn.execute("SELECT * FROM users WHERE id = ?", user_id).fetchone()

    def get_profile_after_update(self, user_id):
        # Read-your-own-write: must go to primary
        with self.primary.connect() as conn:
            return conn.execute("SELECT * FROM users WHERE id = ?", user_id).fetchone()
```

Many ORM frameworks provide native read/write split:
- **Django**: `DATABASES` with `primary` and `replica`, plus a custom router.
- **Rails ActiveRecord**: `role: :writing` vs `role: :reading`.
- **Spring Data**: `@ReadOnly` annotation routes to replica.

#### Proxy-Level Routing

A database proxy inspects query intent and routes automatically:
- **ProxySQL**: parses SQL and routes `SELECT` to replicas, `INSERT/UPDATE/DELETE` to primary.
- **AWS RDS Proxy**: routing support with Aurora.
- **MaxScale (MariaDB)**: read/write split with replica awareness.

```
App → ProxySQL → (SELECT queries) → Replica pool
              → (INSERT/UPDATE/DELETE) → Primary
```

Proxy-level routing is transparent to the application but requires careful handling of transactions (all statements in a transaction must go to the primary).

### Read Replica for Specialized Workloads

**Reporting and Analytics:**
Long-running analytical queries on the primary can starve OLTP queries. Run them on a dedicated replica:
```
Analytics replica:  runs month-end reports, dashboard aggregations
Production traffic: served by primary + regular replicas
```

**Batch Processing:**
Data exports, ETL pipelines, and ML training read from a dedicated replica. No risk of impacting user-facing queries.

**Global Read Distribution:**
Cross-region replicas serve local reads with lower latency:
```
Primary: us-east-1 (writes)
Replica: eu-west-1 (serves European users)
Replica: ap-southeast-1 (serves APAC users)
```

### Replication Lag — The Core Limitation

Replicas are **asynchronous** by default. There is a propagation delay (typically < 100ms, but can spike to seconds under heavy write load) between a write on the primary and its visibility on replicas.

**Consequences:**
- A user creates a post and immediately refreshes the feed — the post may not appear (read from a lagging replica).
- A user changes their email, then tries to log in — the old email works on a lagging replica.
- An order is placed and payment confirmation is checked — the payment record might not be on the replica yet.

**Mitigations:**
1. **Read from primary for a window after a write**: for 30–60 seconds after a user writes, route their reads to primary.
2. **Read your own writes from primary only**: track whether the current session has written recently.
3. **Semi-synchronous replication**: primary waits for at least one replica to confirm before returning. Reduces lag to near-zero at cost of write latency.
4. **Check replication lag before routing**: query `SHOW REPLICA STATUS` (MySQL) or the lag metric from your monitoring system; skip lagging replicas.

---

## Key Trade-offs

| Advantage | Disadvantage |
|---|---|
| Increases total read throughput linearly | Writes are still bottlenecked by primary |
| Isolates analytics/batch from OLTP | Replication lag can cause stale reads |
| Provides a standby for failover | More infrastructure to monitor and maintain |
| Simple to implement (no application sharding) | Primary is still single writer — SPOF for writes |
| Low operational complexity vs sharding | Connection count grows with replica count |

---

## When to use

Read replicas are appropriate when:
- Your database **CPU or throughput is constrained by read queries** (not writes).
- You have heavy **reporting or analytics** queries that interfere with transactional queries.
- You want to **distribute reads geographically** for lower latency in other regions.
- You need a **hot standby** available for failover.
- Read operations **significantly outnumber writes** (typical for content sites, catalogs, feeds).

Read replicas don't help when:
- The bottleneck is **write throughput** → consider sharding.
- The bottleneck is a **single hot row** (all users write to the same counter) → use Redis atomic operations.
- You need **strong read consistency** for all reads → read from primary only.

---

## Common Pitfalls

- **Business-critical reads from replicas**: `SELECT balance FROM accounts` immediately after `UPDATE accounts SET balance = ...` must come from the primary. Blindly routing all SELECTs to replicas causes money/inventory logic errors.
- **Ignoring lag under write load spikes**: during a flash sale or viral event, write load spikes, replication lag can reach seconds. Monitor lag continuously and have a fallback to primary.
- **Not monitoring replica lag**: lag grows silently. Add alerts on replica lag > 1s and alerting policies before it becomes a correctness problem.
- **Using replicas as the only HA mechanism**: a replica takes 30–120s to promote to primary. For HA, use asynchronous replication + automatic failover (Patroni, RDS Multi-AZ). Test failover regularly.

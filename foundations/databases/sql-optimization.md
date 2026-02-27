# SQL Query Optimization

## What it is

**SQL query optimization** is the set of techniques used to improve database query performance — reducing query latency, CPU usage, I/O, and memory consumption. It covers understanding how the database executes queries, identifying bottlenecks, and restructuring queries or schemas to eliminate waste.

At scale, unoptimized queries are one of the most common sources of application bottlenecks.

---

## How it works

### The Query Planner and EXPLAIN

Before optimizing, understand what the database is actually doing. Every RDBMS has a **query planner** that analyzes the SQL statement, consults table statistics, and selects the lowest-cost execution plan.

```sql
-- EXPLAIN shows the plan without executing the query
EXPLAIN SELECT * FROM orders WHERE user_id = 42;

-- EXPLAIN ANALYZE actually runs the query and shows real row counts + timings
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 42;
```

**Reading EXPLAIN output (PostgreSQL):**
```
Seq Scan on orders  (cost=0.00..24521.00 rows=1 width=72)
                                            ^--- estimated rows

Index Scan using idx_orders_user_id on orders
  Index Cond: (user_id = 42)
  (actual time=0.05..0.12 rows=8 loops=1)
  Buffers: shared hit=4
```

Key terms:
- **Seq Scan**: reads every row in the table — only acceptable for small tables or queries that return most rows.
- **Index Scan**: uses an index to locate rows — efficient for selective predicates.
- **Index Only Scan**: satisfied entirely by the index — no heap access (most efficient read path).
- **Nested Loop / Hash Join / Merge Join**: different join algorithms; planner selects based on data size and available indexes.
- **cost=X..Y**: estimated total cost (in arbitrary planner units). Lower is better.

### Index Optimization

Indexes are the single most impactful optimization lever. See [Indexing](indexing.md) for the full deep dive. Key rules for query optimization:

1. **Add indexes on `WHERE`, `JOIN ON`, and `ORDER BY` columns** that appear in slow queries.
2. **Use composite indexes** ordered by equality predicates first, range predicates last.
3. **Use covering indexes** to eliminate heap fetches for read-hot queries.
4. **Remove unused indexes** — they slow down writes without benefiting reads.

```sql
-- Confirm an index is used:
EXPLAIN ANALYZE SELECT id, total FROM orders
WHERE user_id = 42 AND status = 'pending'
ORDER BY created_at DESC;

-- Create optimal composite index for this query:
CREATE INDEX idx_orders_user_status_date
ON orders (user_id, status, created_at DESC);
```

### Avoid SELECT *

`SELECT *` fetches every column — many of which may not be needed. This:
- Increases I/O (wider rows, more disk pages read).
- Prevents covering index optimization (can't do index-only scan if all columns are needed but not indexed).
- Exposes code to breakage when columns are added/renamed.

```sql
-- Bad:
SELECT * FROM users WHERE id = 42;

-- Good:
SELECT id, name, email FROM users WHERE id = 42;
```

### N+1 Query Problem

The N+1 problem occurs when application code executes one query to fetch a list, then one additional query per item in the list:

```python
# Bad — N+1: 1 query for users + N queries for orders
users = db.query("SELECT * FROM users LIMIT 100")
for user in users:
    orders = db.query("SELECT * FROM orders WHERE user_id = ?", user.id)
```

This generates 101 database round-trips instead of 2.

**Fix with a JOIN or batch query:**
```python
# Good — 1 query with JOIN
rows = db.query("""
    SELECT u.id, u.name, o.id AS order_id, o.total
    FROM users u
    LEFT JOIN orders o ON o.user_id = u.id
    WHERE u.id IN (SELECT id FROM users LIMIT 100)
""")

# Or batch: get all user IDs, then one query for all orders
user_ids = [u.id for u in users]
orders = db.query("SELECT * FROM orders WHERE user_id = ANY(?)", user_ids)
```

ORMs often generate N+1 patterns automatically. In Django: `select_related()` / `prefetch_related()`. In ActiveRecord: `.includes()`.

### Pagination

Never use `OFFSET` for deep pagination on large tables. `OFFSET 100000 LIMIT 20` causes the database to scan and discard 100,000 rows on every page request.

**Use keyset pagination (cursor-based) instead:**
```sql
-- Bad: offset pagination — O(  offset) scan
SELECT * FROM posts ORDER BY created_at DESC OFFSET 100000 LIMIT 20;

-- Good: keyset pagination — O(log n) index seek
SELECT * FROM posts
WHERE created_at < :last_seen_created_at   -- cursor from previous page
ORDER BY created_at DESC
LIMIT 20;
```

Keyset pagination is O(log n) because it uses an index range scan from the cursor position, not a full offset scan.

### Query Rewrites

Some query patterns are inefficient but can be rewritten:

**NOT IN vs NOT EXISTS:**
```sql
-- Often slower (may not use index well, NULLs cause unexpected behavior):
SELECT * FROM users WHERE id NOT IN (SELECT user_id FROM banned_users);

-- Faster and correct with NULLs:
SELECT u.* FROM users u
WHERE NOT EXISTS (SELECT 1 FROM banned_users b WHERE b.user_id = u.id);
```

**Correlated subquery → JOIN:**
```sql
-- Slow: executes inner query once per outer row
SELECT u.*, (SELECT COUNT(*) FROM orders WHERE user_id = u.id) AS order_count
FROM users u;

-- Fast: one pass over both tables
SELECT u.*, COUNT(o.id) AS order_count
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
GROUP BY u.id;
```

**Wildcard leading `%` prevents index use:**
```sql
-- Can't use index — full scan:
WHERE name LIKE '%alice%';

-- Can use index — left-anchored:
WHERE name LIKE 'alice%';
```

### Partition Pruning

Large tables can be partitioned on a column (e.g., `created_at` by month). Queries that filter on the partition key will only scan the relevant partition rather than the entire table.

```sql
-- Create a partitioned table:
CREATE TABLE events (
  id BIGINT,
  event_type VARCHAR,
  occurred_at TIMESTAMPTZ
) PARTITION BY RANGE (occurred_at);

CREATE TABLE events_2024_01 PARTITION OF events
  FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

-- This query only scans the 2024-01 partition:
SELECT * FROM events
WHERE occurred_at BETWEEN '2024-01-01' AND '2024-01-31';
```

Partition pruning requires filtering on the partition key in the `WHERE` clause.

### Connection Pooling and Query Overhead Reduction

Every new database connection is expensive (TCP handshake + authentication + memory allocation). Use a connection pool (PgBouncer, ProxySQL) to reuse connections. Connection overhead should not appear in query profiling — if it does, pooling is missing.

### Statistics and Vacuum

The query planner relies on **table statistics** (row counts, column value distribution) to estimate costs. If statistics are stale (table changed significantly since last `ANALYZE`), the planner makes poor choices.

```sql
-- Manually update statistics:
ANALYZE orders;

-- Rebuild table + indexes to reclaim space and reset statistics:
VACUUM ANALYZE orders;
```

Ensure `autovacuum` is enabled and configured appropriately for high-churn tables.

---

## Key Trade-offs

| Optimization | Gain | Cost |
|---|---|---|
| Add index | Faster reads | Slower writes, more storage |
| Covering index | Eliminate heap fetch | Larger index |
| Keyset pagination | O(log n) page load | Requires stable sort key |
| Denormalization | Fewer joins | Write complexity, consistency risk |
| Partition + pruning | Fast scans on large tables | Partition maintenance overhead |

---

## Optimization Workflow

1. **Measure first**: identify the slowest queries via `pg_stat_statements` (PostgreSQL) or slow query log (MySQL).
2. **Run `EXPLAIN ANALYZE`**: understand the actual execution plan.
3. **Look for Seq Scans on large tables**: add indexes.
4. **Look for high row estimates vs actuals**: stale statistics — run `ANALYZE`.
5. **Check for N+1 patterns**: batch queries or use JOINs.
6. **Check for `OFFSET` pagination**: convert to keyset.
7. **Consider partitioning** for very large time-series tables.
8. **Measure again**: confirm the change actually improved the query.

---

## Common Pitfalls

- **Optimizing without measuring**: adding indexes to tables that aren't bottlenecks wastes effort and adds write overhead.
- **Trusting the ORM blindly**: ORMs generate SQL that is often suboptimal or produces N+1 patterns. Log and review the actual SQL generated in development.
- **Ignoring `EXPLAIN` row estimates**: when estimated rows are dramatically wrong, the plan is probably wrong. Fix statistics or add constraints to help the planner.
- **Vacuuming neglect**: tables with heavy UPDATE/DELETE and insufficient autovacuum experience table bloat and planner degradation.
- **Premature query caching**: caching at the application layer is powerful, but it should not replace fundamental query correctness. Fix the query first, cache second.

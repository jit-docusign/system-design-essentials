# Indexing

## What it is

An **index** is an auxiliary data structure that allows a database to locate rows matching a query **without scanning every row in the table**. Indexes trade **write overhead** (index must be maintained on every change) and **storage space** for significantly faster **reads**.

Without an index, every `WHERE`, `JOIN ON`, or `ORDER BY` clause triggers a full table scan — reading every row, which becomes catastrophic at scale.

---

## How it works

### B-Tree Index (Default)

The B-tree (balanced tree) is the default index type in virtually every RDBMS. It keeps values in sorted order, making it efficient for:
- **Equality**: `WHERE email = 'alice@example.com'`
- **Range**: `WHERE created_at BETWEEN '2024-01-01' AND '2024-06-01'`
- **Prefix search**: `WHERE name LIKE 'Ali%'` (only leading wildcard-free)
- **ORDER BY** and **GROUP BY** on indexed columns

```
B-Tree structure (balanced, O(log n) lookup):

                   [50]
                 /       \
           [25]              [75]
          /    \            /    \
       [10]   [40]       [60]   [90]
```

Every leaf node contains the indexed value and a **pointer** (page + row offset) to the actual data row (called a "heap" row). A query walks the B-tree in O(log n) levels, then fetches the actual row.

### Hash Index

Stores an exact hash of the indexed column. Only useful for **equality lookups**. Not useful for range queries or ordering.

- **Pros**: O(1) lookup for exact equality.
- **Cons**: Doesn't support range queries, `LIKE`, or sorting. Rebuilt on crash in older PostgreSQL versions.

```sql
CREATE INDEX ON sessions USING HASH (session_token);
```

### Covering Index

A covering index includes all the columns needed to satisfy a query, meaning the database **never needs to access the main table** (heap fetch avoided). This is called an **index-only scan**.

```sql
-- Query: SELECT name, email FROM users WHERE status = 'active'
-- Covering index includes all three columns:
CREATE INDEX idx_users_status_covering ON users (status) INCLUDE (name, email);
```

The `INCLUDE` columns are stored in the leaf nodes of the B-tree but not used for sorting — this avoids unnecessary heap fetches for wide rows.

### Composite (Multi-Column) Index

An index on multiple columns. **Column order matters critically** — the index is only usable if the query filters on a **left-prefix** of the indexed columns.

```sql
CREATE INDEX idx_orders_user_status ON orders (user_id, status, created_at);
```

This index can satisfy:
- `WHERE user_id = 1` ✅
- `WHERE user_id = 1 AND status = 'paid'` ✅
- `WHERE user_id = 1 AND status = 'paid' AND created_at > '2024-01-01'` ✅
- `WHERE status = 'paid'` ❌ (not a left-prefix; full table scan)
- `WHERE user_id = 1 AND created_at > '2024-01-01'` ⚠️ (partially usable — uses `user_id` range, has to scan for `created_at`)

**Rule of thumb**: Put high-cardinality equality columns first, then range columns, then columns used only for covering.

### Partial Index

An index that only covers a **subset of rows** matching a condition. Smaller, faster to build, avoids indexing rows that would never be queried.

```sql
-- Only index orders that aren't completed — common for "active work" queries
CREATE INDEX idx_orders_active ON orders (user_id, created_at)
WHERE status != 'completed';
```

### Expression Index

Index on a computed expression rather than a raw column.

```sql
-- Allows case-insensitive email lookups efficiently
CREATE INDEX idx_users_email_lower ON users (LOWER(email));
-- Query can now use the index:
SELECT * FROM users WHERE LOWER(email) = 'alice@example.com';
```

### Full-Text Search Index

For searching within text content. Most RDBMS have built-in full-text index types:
- PostgreSQL: `CREATE INDEX ON documents USING GIN (to_tsvector('english', body))`
- MySQL: `CREATE FULLTEXT INDEX ON articles (title, body)`

Better replaced by dedicated search engines (Elasticsearch, OpenSearch) for serious search workloads.

### Index Cardinality

**Cardinality** = number of distinct values in a column. High cardinality = index is very selective (fewer rows per value).

- **High cardinality** (good for indexing): `user_id`, `email`, `UUID`
- **Low cardinality** (poor index efficiency): `status` with 3 values, `is_active` boolean

Indexing a boolean column that maps to millions of rows per value is often counterproductive — the database still has to read huge amounts of data after the index lookup and may prefer a full scan anyway.

### How the Query Planner Uses Indexes

The query planner (optimizer) estimates the cost of different access paths and chooses the cheapest:

```sql
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 42 AND status = 'pending';
```

```
Index Scan using idx_orders_user_status on orders
  Index Cond: ((user_id = 42) AND (status = 'pending'))
  Rows: 12  (estimated: 10)
  Actual time: 0.15ms
```

If the table has been vacuumed, statistics are current, and row estimates are close to actual, the planner chooses well. Stale statistics (from high-churn tables not vacuumed) cause bad plan choices.

---

## Key Trade-offs

| Factor | Impact |
|---|---|
| More indexes | Faster reads, slower writes (every INSERT/UPDATE/DELETE updates all indexes) |
| Covering index | Eliminates heap fetch, biggest read speedup |
| Too many indexes | Increased storage, worse write performance, slower bulk loads |
| Low cardinality index | Index overhead without much read benefit |
| Missing index on JOIN | Join becomes O(n×m) instead of O(n log m) |

---

## When to use (which index type)

| Workload | Best index choice |
|---|---|
| Equality + range queries | B-tree (default) |
| Exact equality only | Hash |
| Read-heavy with narrow SELECT columns | Covering index |
| Sparse "hot" subset of rows | Partial index |
| Case-insensitive or computed predicates | Expression index |
| Full-text keyword search | GIN/GiST or dedicated search engine |
| Geospatial queries | GiST (PostGIS), R-tree |
| Array/JSONB containment | GIN |

---

## Common Pitfalls

- **Not running `EXPLAIN ANALYZE`**: guessing which indexes to add without reading the query plan is wasteful. Always profile before adding indexes.
- **Indexing every column**: each index adds write overhead. Only index columns that appear in `WHERE`, `JOIN ON`, or `ORDER BY` in actual slow queries.
- **Unused indexes**: over time, indexes accumulate. Periodically review `pg_stat_user_indexes` (PostgreSQL) to identify unused indexes and drop them.
- **Wrong column order in composite index**: `(status, user_id)` and `(user_id, status)` are very different. Model based on actual query predicates.
- **Not accounting for write-heavy tables**: on a table with thousands of writes per second, every additional index adds latency to each write. Profile write performance after adding indexes.
- **`LIKE '%keyword%'` doesn't use B-tree**: a leading wildcard prevents index use. Use full-text indexes or search engines for infix search.
- **Index bloat**: after many updates/deletes, B-tree pages can become sparse. Rebuild or reorganize indexes periodically on write-heavy tables.

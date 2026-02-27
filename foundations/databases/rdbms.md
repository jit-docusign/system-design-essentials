# Relational Databases (RDBMS)

## What it is

A **Relational Database Management System (RDBMS)** stores data in structured **tables** with defined schemas, enforces **relationships** between tables via foreign keys, and provides rich query capabilities through **SQL (Structured Query Language)**. It guarantees **ACID** properties for all transactions.

Relational databases are the default persistence choice for most applications because they provide correctness guarantees, well-understood query semantics, and decades of operational tooling.

---

## How it works

### Core Data Model

Data is organized into **tables** (relations). Each table has:
- **Columns (attributes)**: defined with a data type and constraints.
- **Rows (tuples)**: individual records.
- **Primary key**: uniquely identifies each row.
- **Foreign keys**: reference the primary key of another table, enforcing referential integrity.

```sql
CREATE TABLE users (
  id         BIGINT PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
  email      VARCHAR(255) NOT NULL UNIQUE,
  name       VARCHAR(100) NOT NULL,
  created_at TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE orders (
  id         BIGINT PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
  user_id    BIGINT NOT NULL REFERENCES users(id),
  total      DECIMAL(10, 2) NOT NULL,
  status     VARCHAR(20) NOT NULL DEFAULT 'pending',
  created_at TIMESTAMPTZ DEFAULT now()
);
```

### Normalization

Normalization is the process of structuring tables to **minimize data redundancy** and **prevent update anomalies**:

- **1NF**: Each column holds atomic values (no arrays in a column); each row is unique.
- **2NF**: No partial dependency — every non-key column depends on the entire primary key (relevant for composite keys).
- **3NF**: No transitive dependency — non-key columns depend only on the primary key, not on other non-key columns.

**Example of transitive dependency (violates 3NF):**
```
order_id → customer_id → customer_city
```
`customer_city` depends on `customer_id`, not on `order_id`. Move `customer_city` to a `customers` table.

The goal of normalization is to ensure each fact is stored **once** — updates change exactly one row rather than many.

### Joins

Joins combine rows from multiple tables based on a related column. The most common:

```sql
-- INNER JOIN: rows that match in both tables
SELECT u.name, o.total
FROM users u
INNER JOIN orders o ON o.user_id = u.id;

-- LEFT JOIN: all rows from left table; NULLs for unmatched right rows
SELECT u.name, COUNT(o.id) AS order_count
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
GROUP BY u.id;
```

Joins are powerful but require indexes on join columns to be performant. Large joins on unindexed tables are the most common source of slow queries.

### Indexes

Indexes are auxiliary data structures that speed up read queries at the cost of write overhead (index must be updated on every insert/update/delete). See [Indexing](indexing.md) for a deep dive.

### Transactions and ACID

Every multi-step operation in a relational database is wrapped in a **transaction**:

```sql
BEGIN;
  UPDATE accounts SET balance = balance - 500 WHERE id = 1;
  UPDATE accounts SET balance = balance + 500 WHERE id = 2;
  INSERT INTO transfers (from_id, to_id, amount) VALUES (1, 2, 500);
COMMIT;
-- If any step fails: ROLLBACK automatically reverts all changes
```

See [ACID Properties](../core-concepts/acid.md) and [Transactions & Locking](transactions-locking.md) for details.

### Storage Engine Internals

Modern RDBMS (PostgreSQL, MySQL/InnoDB) use:
- **B-tree index structures** for efficient range queries and sorted access.
- **MVCC (Multi-Version Concurrency Control)**: older row versions are kept for concurrent readers; writers create new versions without blocking readers.
- **WAL (Write-Ahead Log)**: all changes are logged before being applied to data pages — ensures durability and enables point-in-time recovery.
- **Buffer pool**: a memory cache of frequently accessed data pages.
- **Vacuum / autovacuum** (PostgreSQL): clean up dead row versions from MVCC that are no longer needed.

### Scaling Characteristics

The default scaling path for RDBMS:
1. **Vertical scaling**: increase server CPU/RAM/SSD. Most pragmatic first step.
2. **Connection pooling** (PgBouncer, ProxySQL): databases have limited connection capacity; pooling multiplexes many application connections into fewer database connections.
3. **Read replicas**: offload SELECT-heavy workloads to replicas.
4. **Caching**: cache hot query results in Redis/Memcached to reduce database load.
5. **Sharding**: horizontal partition the dataset — more complex, discussed in [Sharding](sharding.md).

---

## Key Trade-offs

| Advantage | Disadvantage |
|---|---|
| Strong ACID guarantees | Horizontal write scaling is hard |
| Rich SQL query language with joins | Schema migrations require careful planning |
| Mature ecosystem and tooling | Vertical scaling has an upper limit |
| Referential integrity enforced by DB | Object-relational impedance mismatch |
| Well-understood operational behavior | Connection count limits at high concurrency |

---

## When to use

Choose a relational database (over NoSQL) when:
- Your data has a clear relational structure with consistent schemas.
- You need ACID transactions with multiple tables/rows.
- You need ad-hoc queries with complex joins and aggregations.
- You need foreign key constraints and referential integrity.
- You're ok with vertical scaling and read replicas as your primary scaling model.

Common use cases: user accounts, financial transactions, e-commerce orders, product catalogs, CMS content, analytics with complex reporting.

---

## Common Pitfalls

- **Not using transactions**: multi-step operations outside transactions can leave data in an inconsistent state if interrupted mid-way.
- **Over-normalizing**: fully normalizing a schema for an OLAP analytical workload causes excessive joins and poor query performance. Denormalize for read-heavy reporting.
- **Missing indexes on join columns and WHERE predicates**: the most common cause of slow queries. Run `EXPLAIN ANALYZE` to identify missing indexes.
- **Unbounded queries**: `SELECT * FROM orders` without a `LIMIT` on a table with 100M rows will kill your database. Always paginate.
- **Too many connections**: databases (especially PostgreSQL) have a hard limit on concurrent connections. Not using connection pooling causes connection exhaustion at scale.
- **Schema changes without care**: `ALTER TABLE ADD COLUMN` can lock the table on some databases. Use online schema change tools (pt-online-schema-change, pg_repack) for large tables in production.

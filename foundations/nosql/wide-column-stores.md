# Wide-Column Stores

## What it is

A **wide-column store** (also called a **column-family database**) organizes data in a two-level structure: **rows** identified by a row key, and **columns** that are grouped into **column families**. Unlike relational databases where every row has the same fixed columns, wide-column stores allow each row to have a **different set of columns**, and tables can have millions of columns.

Wide-column stores are designed for **high write throughput**, **time-series data**, **sparse data**, and **massive scale** across distributed clusters.

**Common systems**: Apache Cassandra, HBase (Hadoop), Google Bigtable, Amazon Keyspaces (managed Cassandra), ScyllaDB.

---

## How it works

### Data Model

Think of a wide-column store as a **nested sorted map**:

```
Table: user_activity

Row Key (user_id)  | Column Family: events
                   | timestamp:type           | value
───────────────────┼─────────────────────────┼──────────────────────
"user:1001"        | 1720000001:click         | {"page": "/cart"}
                   | 1720000045:purchase      | {"order_id": "ord-99"}
                   | 1720000120:logout        | {}
───────────────────┼─────────────────────────┼──────────────────────
"user:1002"        | 1720000003:login         | {"ip": "1.2.3.4"}
                   | 1720000099:click         | {"page": "/home"}
```

Key characteristics:
- **Row key**: the primary access path — all queries must specify a row key (or range of keys).
- **Column qualifier** = column family + column name: dynamically defined per row.
- **Sparse storage**: columns that have no value simply don't exist — no NULL storage.
- Columns within a row are **sorted** by name — range scans within a row are efficient.

### Cassandra Specifics

Cassandra is the most widely used wide-column store. It uses a **ring topology** with consistent hashing and no single point of failure.

**CQL (Cassandra Query Language)** is SQL-like but has important restrictions driven by the distributed data model:

```sql
-- Create table: partition key + clustering key
CREATE TABLE user_events (
  user_id   UUID,
  occurred_at TIMESTAMP,
  event_type TEXT,
  payload   TEXT,
  PRIMARY KEY (user_id, occurred_at)
) WITH CLUSTERING ORDER BY (occurred_at DESC);

-- Insert (upsert — no separate INSERT vs UPDATE in Cassandra)
INSERT INTO user_events (user_id, occurred_at, event_type, payload)
VALUES (uuid(), toTimestamp(now()), 'click', '{"page":"/cart"}');

-- Query — MUST filter on full partition key
SELECT * FROM user_events
WHERE user_id = 550e8400-e29b-41d4-a716-446655440000
  AND occurred_at > '2024-01-01'
LIMIT 100;

-- Disallowed — no partition key specified (full table scan)
-- SELECT * FROM user_events WHERE event_type = 'purchase';  -- FAILS
```

### Partition Key and Clustering Key

The **primary key** in Cassandra has two components:

```
PRIMARY KEY (partition_key, clustering_key_1, clustering_key_2)
```

- **Partition key**: determines which node(s) hold the row (`hash(partition_key)` → node).
- **Clustering key**: determines the sort order of rows within a partition. Enables efficient range queries **within a partition**.

Designing the partition key is the most critical schema decision:
- **Too narrow** (e.g., `user_id` for a billion users): too many small partitions → many small read operations, hot single-user partitions (if one user has millions of events, one partition gets very large → "wide partition" problem).
- **Too broad** (e.g., `date` for daily events): all writes for a given day hit one partition → hot spot.

**Bucket the partition key** to balance: `(user_id, bucket)` where `bucket = event_timestamp / 30_days` distributes a user's events across monthly partitions.

### Cassandra Consistency Levels

Cassandra is an **AP system** by default but offers **tunable consistency**:

| Write Consistency | Meaning |
|---|---|
| `ONE` | Write confirmed by any 1 replica |
| `QUORUM` | Write confirmed by majority of replicas |
| `ALL` | Write confirmed by all replicas |

| Read Consistency | Meaning |
|---|---|
| `ONE` | Read from any 1 replica (may be stale) |
| `QUORUM` | Read from majority of replicas (most recent value) |
| `LOCAL_QUORUM` | Quorum within the local datacenter (cross-DC latency avoided) |

Achieving strong consistency: use **QUORUM** writes + **QUORUM** reads when replication factor = 3. `W + R > N` (2 + 2 > 3).

### Write Path (Why Cassandra is Write-Optimized)

1. Write is appended to the **commit log** (sequential disk write — fast).
2. Write is stored in the **MemTable** (in-memory).
3. MemTable periodically flushed to an immutable **SSTable** on disk.
4. **Compaction** merges SSTables, removing old versions and tombstones.

All writes are sequential appends — no random disk I/O. This makes Cassandra exceptionally write-efficient.

### Read Path

1. Check MemTable.
2. Check Bloom filter (probabilistic — is the key likely in this SSTable?).
3. Check SSTable key index.
4. Read from SSTable page cache or disk.

Reads in Cassandra are more expensive than writes because data may be spread across multiple SSTables. Read-heavy workloads should consider compaction strategy and read repair.

---

## Key Trade-offs

| Advantage | Disadvantage |
|---|---|
| Extremely high write throughput | Queries MUST be built around the partition key |
| Linear horizontal scalability | No joins — denormalize by query pattern |
| No single point of failure | Limited secondary index support |
| Tunable consistency | Complex data modeling — must design for access patterns |
| Handles very wide rows (millions of columns) | Schema redesign required when access patterns change |
| Good for time-series with efficient range scans within partitions | Aggregations (COUNT, SUM) are expensive |

---

## When to use

Wide-column stores excel at:
- **Time-series data**: IoT sensor data, user activity logs, financial tick data.
- **High write throughput with linear scalability**: when write rates exceed what a single RDBMS primary can handle.
- **Known, stable query patterns**: you always query by user ID + time range, not arbitrary filters.
- **Sparse data**: rows where most columns have no value (wide-column stores store only present columns).
- **Multi-region, multi-datacenter deployments**: Cassandra natively supports multi-datacenter replication.

---

## Common Pitfalls

- **Modeling like a relational database**: Cassandra's primary constraint is that queries must filter on the partition key. Trying to design second-normal-form schemas leads to full-cluster scans that destroy performance.
- **Unbounded partition growth ("wide partition" problem)**: a partition that accumulates millions of rows (e.g., all events for a high-volume user) becomes slow to read and scan. Time-bucket partition keys to bound partition size.
- **Overusing ALLOW FILTERING**: the `ALLOW FILTERING` directive in CQL forces a full cluster scan when the query doesn't specify a partition key. Never use this in production; it signals a data model problem.
- **Ignoring tombstones**: deletions in Cassandra create tombstones (markers, not actual deletes). On tables with many deletes (e.g., TTL-heavy data), tombstone accumulation degrades read performance until compaction clears them.
- **Wrong compaction strategy**: `SizeTieredCompactionStrategy` (write-heavy), `LeveledCompactionStrategy` (read-heavy), `TimeWindowCompactionStrategy` (time-series). Using the wrong strategy for the workload causes excessive read amplification.

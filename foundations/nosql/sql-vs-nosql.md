# SQL vs NoSQL

## What it is

The SQL vs NoSQL decision is one of the most common — and most nuanced — questions in system design. It is not a binary choice but a spectrum: the right database depends on **access patterns, consistency requirements, scale, team familiarity, and existing system constraints**.

This guide provides a structured framework for making the decision rather than advocating for either side.

---

## Side-by-Side Comparison

| Dimension | SQL (RDBMS) | NoSQL (varies by type) |
|---|---|---|
| **Data model** | Tables with rows and fixed columns | Key-value, document, wide-column, graph — varies |
| **Schema** | Strict, strongly typed | Flexible / schemaless |
| **Relationships** | Native joins, foreign keys | Application-level or embedded |
| **Transactions** | Full multi-table ACID | Varies: none / single-document / tunable |
| **Query language** | Standardized SQL | DB-specific (CQL, MQL, Cypher, PromQL) |
| **Consistency** | Strong consistency by default | Often eventual; some offer tunable consistency |
| **Write scaling** | Hard (sharding requires app changes) | Designed for horizontal write scaling |
| **Read scaling** | Read replicas | Native horizontal scaling |
| **Schema evolution** | Migrations (ALTER TABLE) | Fields can be added without migration |
| **Operational maturity** | Decades of tooling, monitoring, expertise | Varies; newer systems have less tooling |
| **Ad-hoc queries** | Excellent (any SELECT/GROUP BY/JOIN) | Limited to designed access patterns |

---

## Decision Framework

### Step 1: Understand the Access Patterns

**Questions to ask:**
- How will data be queried? By a single key? By multiple attributes? With full-text search? By relationship traversal?
- Are queries known and stable, or ad-hoc and evolving?
- What is the read-to-write ratio?
- Will you need aggregations (COUNT, SUM, AVG) across the dataset?

| Access pattern | Best fit |
|---|---|
| Lookup by primary key | Key-value, any DB |
| Query by multiple attributes + joins | RDBMS |
| Flexible document retrieval by any field | Document store |
| Time-range aggregation on metrics | Time-series DB |
| Friend-of-a-friend traversal | Graph DB |
| Full-text search with ranking | Search engine |
| High-throughput append with time-ordered scan | Wide-column |

### Step 2: Assess Consistency Requirements

| Requirement | Choice |
|---|---|
| Financial transactions, order processing | RDBMS with ACID |
| User sessions, cache | Eventual consistency acceptable |
| Social feed, analytics | Eventual consistency acceptable |
| Inventory reservation (race conditions matter) | RDBMS or strong-consistency store |
| Configuration, coordination | CP system (etcd, ZooKeeper) |

If your business logic depends on "once committed, always visible to all readers immediately", you need strong consistency — which most NoSQL systems don't provide by default.

### Step 3: Assess Scale Requirements

```
Write throughput:
  < 10K writes/sec    → RDBMS with vertical scaling
  10K–100K writes/sec → RDBMS + sharding OR Cassandra/DynamoDB
  > 100K writes/sec   → Wide-column (Cassandra), purpose-built NoSQL

Data volume:
  < 100GB             → Single RDBMS (fits on one node)
  100GB–10TB          → RDBMS + sharding or NoSQL
  > 10TB              → Distributed store

Read throughput:
  Any scale           → Read replicas + caching first
  Very high with simple KV patterns → DynamoDB, Redis
```

### Step 4: Assess Schema Flexibility Needs

- Is the schema known and stable? → RDBMS (strict schema is a feature, not a limitation)
- Is the schema evolving rapidly during development? → Document store (faster iteration)
- Is each record highly heterogeneous (different attributes per item)? → Document store or wide-column
- Is schema validation critical for data integrity? → RDBMS (enforced) or document DB with schema validation

### Step 5: Consider Operational Factors

- **Team expertise**: a team experienced with PostgreSQL will outperform a team learning Cassandra. Familiarity matters.
- **Managed services**: DynamoDB, MongoDB Atlas, and managed PostgreSQL (RDS, Cloud SQL) reduce ops burden significantly.
- **Existing infrastructure**: adding a new database type means new monitoring, new backup procedures, new expertise.

---

## Common System Design Patterns

### RDBMS for Source of Truth + NoSQL for Access Patterns

Most production systems use **polyglot persistence**: choose the right storage medium for each use case rather than forcing one database to do everything.

```
Primary store (writes, ACID operations)   → PostgreSQL
Search                                    → Elasticsearch
Caching                                   → Redis
Session store                             → Redis
Event/activity stream                     → Cassandra or Kafka
Analytics                                 → BigQuery / Redshift
```

### The CQRS Pattern

**Command Query Responsibility Segregation** (CQRS) formalizes the separation:
- **Command side** (writes): go to the primary relational store.
- **Query side** (reads): materialized views in a read-optimized NoSQL store.

Changes propagate from the command side to the query side asynchronously (via events). This allows each side to be optimized independently.

### Choosing in Practice

When selecting a storage technology for a new system, default to:
- **PostgreSQL/MySQL** unless there's a clear access-pattern reason not to.
- **Redis** for caching and sessions.
- **Cassandra/DynamoDB** when you explicitly need write scale (chat messages, activity feeds, IoT).
- **Elasticsearch** when you need full-text search.
- **Time-series DB** when you need metrics.

Make the choice explicit, anchor it in concrete access patterns, and ensure the trade-offs are understood by everyone who will operate the system.

---

## Key Trade-offs Summary

| Factor | Choose SQL | Choose NoSQL |
|---|---|---|
| Data model | Relational, joins required | Hierarchical, graph, time-series, search |
| Consistency | Strong ACID required | Eventual consistency acceptable |
| Schema | Stable, known schema | Evolving, heterogeneous schema |
| Scale | Moderate write load | Very high write load, linear scale |
| Query flexibility | Rich ad-hoc queries | Fixed, known access patterns |
| Operational maturity | Strongly preferred | Acceptable when justified |
| Transaction scope | Multi-table ACID required | Single entity atomicity sufficient |

---

## Common Pitfalls

- **Defaulting to NoSQL because it's "modern"**: NoSQL databases solve specific problems. SQL solves a broader set of problems reliably. Don't adopt NoSQL without a clear justification.
- **Assuming NoSQL is always faster**: Redis is faster than PostgreSQL for a key lookup, but PostgreSQL with a covering index on an in-memory table is comparably fast. Benchmark your specific workload.
- **Not planning for the JOIN problem**: when moving from SQL to a document or key-value store, application code must handle all the relationship resolution SQL handled. This is not "free" — it adds latency and complexity.
- **Underestimating migration complexity**: switching database paradigms mid-project is painful. Make the right choice early, or plan for a parallel migration period.
- **Choosing a database for its logo / hype**: Cassandra, MongoDB, and Elasticsearch have all been misapplied as general-purpose stores. Each has a specific sweet spot.

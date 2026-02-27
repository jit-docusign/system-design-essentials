# NoSQL Databases — Overview

## What it is

**NoSQL** ("Not Only SQL") is a broad category of database systems that depart from the traditional relational model. Rather than a single paradigm, NoSQL encompasses several distinct data models optimized for specific access patterns, scale requirements, or flexibility needs that relational databases don't serve well by default.

NoSQL databases typically sacrifice some relational features (strict schema, complex joins, full ACID guarantees) in exchange for **horizontal scaling**, **flexible schemas**, **high write throughput**, or **purpose-built query semantics**.

---

## How it works

### Why NoSQL Emerged

Relational databases scale vertically well, but horizontal write scaling is hard — sharding an RDBMS requires significant application-level complexity. As web-scale applications emerged (Facebook, Google, Amazon), teams built custom storage systems optimized for their specific workloads:
- Google's **Bigtable** (wide-column) → inspired HBase, Cassandra
- Amazon's **Dynamo** (key-value) → inspired Cassandra, DynamoDB
- These systems prioritized availability and partition tolerance over consistency (AP systems in CAP theorem)

### The Core NoSQL Data Models

| Model | How data is structured | Best for |
|---|---|---|
| **Key-Value** | Opaque value looked up by a unique key | Sessions, caching, simple lookups |
| **Document** | JSON/BSON documents with flexible schema | User profiles, content, product catalogs |
| **Wide-Column** | Tables with rows and dynamic named column families | Time-series, high-write analytics, IoT |
| **Graph** | Nodes and edges with properties | Social graphs, fraud detection, recommendations |
| **Time-Series** | Optimized for series of timestamped values | Metrics, monitoring, financial data |
| **Search Engine** | Inverted index optimized for full-text search | Search, autocomplete, faceted filtering |

Each has its own file in this section with detailed coverage.

### Common NoSQL Characteristics

**1. Schema Flexibility**
Most NoSQL databases are schema-less or schema-optional. Documents in a collection can have different fields:
```json
// User A
{"id": 1, "name": "Alice", "email": "alice@example.com"}

// User B — different fields, valid in the same collection
{"id": 2, "name": "Bob", "email": "bob@example.com", "phone": "+1-555-0100", "tier": "premium"}
```
This accelerates development (no migrations for adding fields) but can lead to inconsistent data if not managed.

**2. Horizontal Scaling by Design**
NoSQL databases are typically designed from the ground up to shard across many nodes. DynamoDB, Cassandra, and MongoDB all distribute data automatically across a cluster.

**3. Denormalized Data Models**
Without joins, NoSQL applications embed related data together (documents) or model data specifically for the access patterns (wide-column). See [Denormalization](../databases/denormalization.md).

**4. Eventual Consistency (Often)**
Many NoSQL systems trade strong consistency for availability and partition tolerance (AP in CAP). Data written to one node may take milliseconds to propagate to others. Some NoSQL databases offer tunable consistency levels (Cassandra, DynamoDB).

### Consistent Hashing and Ring Topology

Many NoSQL databases (Cassandra, DynamoDB) use consistent hashing to distribute data across nodes with minimal resharding on cluster changes. See [Load Balancing Algorithms — Consistent Hashing](../traffic/load-balancing-algorithms.md).

### When to Choose NoSQL Over SQL

| Signal | Consider NoSQL |
|---|---|
| Flexible/evolving schema | Document store |
| Very high write throughput (millions/sec) | Wide-column (Cassandra) or key-value |
| Simple key-based lookups, no joins | Key-value store |
| Graph traversal queries | Graph database |
| Metrics, monitoring data | Time-series DB |
| Full-text search with ranking | Search engine |
| Data size too large to fit single RDBMS node | Any horizontally scalable NoSQL |

---

## Key Trade-offs: SQL vs NoSQL Summary

| Dimension | SQL (RDBMS) | NoSQL |
|---|---|---|
| Schema | Strict, enforced | Flexible or optional |
| Transactions | Full ACID | Varies; often eventual or limited |
| Joins | Native SQL joins | Application-level or pre-joined data |
| Scaling | Vertical + read replicas + hard sharding | Horizontally scalable by design |
| Query flexibility | Rich ad-hoc SQL | Limited to primary access patterns |
| Maturity | Decades, very mature | Varies per system |
| Operations | Unified tooling | Diverse ecosystem, multiple tools |

---

## The "Polyglot Persistence" Pattern

Modern large-scale systems don't choose one database for everything — they use the right tool for each data access pattern:

```
User Service         → PostgreSQL (RDBMS — profiles, authentication)
Session Store        → Redis (key-value — sessions, rate limit counters)
Product Catalog      → MongoDB (document — flexible product schemas)
Activity Feed        → Cassandra (wide-column — time-series, high write)
Search               → Elasticsearch (search engine — full-text, facets)
Analytics            → BigQuery (columnar warehouse — aggregations)
Social Graph         → Neo4j (graph — friend-of-a-friend queries)
Metrics/Monitoring   → InfluxDB or Prometheus (time-series)
```

The trade-off: each system adds operational complexity. Only add a new database type when the capability gap justifies the overhead.

---

## Common Pitfalls

- **Choosing NoSQL for flexibility, not access patterns**: "document database just feels easier" is not an architecture decision. Model data based on how it will be queried.
- **Expecting SQL-level consistency from an AP system**: if you need strong consistency, choose CP NoSQL (MongoDB with majority write concern) or use a relational database.
- **No schema governance in document stores**: schema-less doesn't mean no schema. Without discipline, document collections become chaotic and difficult to query reliably.
- **Modeling a relational problem in a document store**: if you find yourself doing many joins in application code across multiple document types, a relational database may be more appropriate.
- **Adding too many data stores**: each database system requires expertise, monitoring, backup, and ops. Start with fewer systems and add only when justified.

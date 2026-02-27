# Data Lake vs Data Warehouse

## What it is

**Data Lake** and **Data Warehouse** are two architectural patterns for storing and analyzing large volumes of data at scale. They differ in structure, governance, cost, query patterns, and use cases — and modern systems often combine both.

**Data Lake**: a centralized repository that stores **raw, unprocessed data in its native format** (structured, semi-structured, unstructured). Schema is defined when reading (schema-on-read). Storage is cheap object storage (S3, GCS, HDFS).

**Data Warehouse**: a structured, curated repository optimized for **analytical SQL queries**. Data is cleaned, transformed, and loaded in a predefined schema (schema-on-write). Optimized for fast query performance. Examples: Snowflake, BigQuery, Redshift, ClickHouse.

---

## How it works

### Data Lake Architecture

```
Data Sources (raw ingestion):
  ├── Application databases (CDC via Debezium)
  ├── Event streams (Kafka → S3 via Flink/Spark)
  ├── Log files (Fluentd → S3)
  ├── Third-party APIs
  └── IoT / sensor data

                    ▼

        Data Lake (S3 / GCS / ADLS)
        ┌───────────────────────────────────────┐
        │  /raw/         (unprocessed, as-is)   │
        │  /curated/     (cleaned, partitioned)  │
        │  /aggregated/  (pre-computed)          │
        └───────────────────────────────────────┘
              │                    │
    ┌─────────┘               ┌────┘
    ▼                         ▼
Exploration              Batch Processing
(Athena, Spark,          (Spark, dbt, Flink)
 Presto, Hive)                │
                              ▼
                          Data Warehouse
```

**Key properties:**
- Store everything, transform later (ELT: Extract-Load-Transform).
- Data is immutable and append-only (raw zone preserved).
- Cheap: ~$0.023/GB/month on S3 vs $500/TB/month for some warehouses.
- Flexible: different teams can query the same raw data with different schemas.

### Data Warehouse Architecture

**Schema-on-write**: data is cleaned, transformed, and loaded in a well-defined relational schema before being queryable.

```
ETL Pipeline:
  Extract  → pull data from sources
  Transform → clean, join, aggregate, apply business rules
  Load     → insert into warehouse tables (normalized or denormalized)

Warehouse Schema:
  Star Schema (most common):
    FACT_ORDERS         ← central fact table (transactions, events)
      order_id
      user_id           → DIM_USERS (user attributes)
      product_id        → DIM_PRODUCTS (product attributes)
      order_date_id     → DIM_DATE (date attributes for time-based analysis)
      total_amount
      quantity
  
  Result: fast analytical queries with simple JOINs:
    SELECT d.year, p.category, SUM(f.total_amount)
    FROM FACT_ORDERS f
    JOIN DIM_DATE d ON f.order_date_id = d.date_id
    JOIN DIM_PRODUCTS p ON f.product_id = p.product_id
    GROUP BY d.year, p.category
```

**Columnar storage**: warehouses store data column-by-column (not row-by-row). A query selecting 3 columns from 1 billion rows reads only 3 columns' worth of data — massive I/O savings.

```
Row storage:
  Row 1: [user_id=1, name="Alice", email="alice@x.com", age=30, city="NY", ...]
  Row 2: [user_id=2, name="Bob",   email="bob@x.com",   age=25, city="LA", ...]

Column storage:
  user_id: [1, 2, 3, 4, ...]
  name:    ["Alice", "Bob", "Carol", ...]
  city:    ["NY", "LA", "NY", ...]

Query: SELECT city, COUNT(*) FROM users GROUP BY city
  → reads ONLY the "city" column — skips user_id, name, email, age
```

### Data Lakehouse

The modern **Lakehouse** pattern combines lake flexibility with warehouse performance by adding a metadata/transactional layer on top of object storage.

```
Object Storage (S3 / GCS)
     ↑
Delta Lake / Apache Iceberg / Apache Hudi   ←── ACID transactions, schema evolution,
     ↑                                           time travel, efficient updates/deletes
SQL Engines (Spark, Trino, Databricks, Athena)
     ↑
BI Tools (Tableau, Looker, Power BI)
```

**Delta Lake features** (on top of Parquet in S3):
- ACID transactions: concurrent writes are safe.
- Time travel: query data as of a past timestamp.
- Schema enforcement and evolution.
- Efficient upserts and deletes (important for GDPR right-to-erasure).
- Z-ordering: co-locate related data for faster query pruning.

```python
# Delta Lake: UPSERT (MERGE) in Spark
from delta.tables import DeltaTable

delta_table = DeltaTable.forPath(spark, 's3://data-lake/delta/orders/')

delta_table.alias('target').merge(
    source=new_records.alias('source'),
    condition='target.order_id = source.order_id'
).whenMatchedUpdateAll() \
 .whenNotMatchedInsertAll() \
 .execute()
```

### Comparison: Data Lake vs Data Warehouse vs Lakehouse

| Dimension | Data Lake | Data Warehouse | Lakehouse |
|---|---|---|---|
| Schema | On-read (flexible) | On-write (rigid) | On-write with evolution |
| Data format | Raw (JSON, CSV, logs, Parquet) | Curated (columnar, dimensional) | Open (Parquet + Delta/Iceberg) |
| Storage cost | Very low ($0.02/GB) | High ($5–50/GB effective) | Low ($0.02/GB) |
| Query performance | Slow (no indexing, full scan) | Fast (columnar, partitioned) | Fast (Z-order, partition pruning) |
| ACID transactions | No | Yes | Yes |
| ML / data science | Excellent (raw data) | Poor (pre-aggregated, no raw) | Excellent |
| BI / reporting | Poor to moderate | Excellent | Excellent |
| Data freshness | Near-real-time ingest | Minutes to hours (ETL) | Near-real-time (streaming upserts) |
| Tool ecosystem | Spark, Flink, Hive, Presto | SQL, Tableau, Looker | SQL + Spark + BI tools |

### Lambda Architecture (Batch + Stream for Data)

```
Data Sources
    │
    ├──► Speed Layer (Kafka + Flink)  → real-time views (last 1h)
    │
    └──► Batch Layer (Spark on S3)   → batch views (full history, overnight)
    
         Serving Layer (warehouse)   → merges batch and real-time views
```

Modern systems increasingly use **Kappa architecture** — only a streaming layer — by making the stream engine replayable (Kafka's durable log) and capable of historical batch reprocessing.

---

## Key Trade-offs

### Data Lake
| Advantage | Disadvantage |
|---|---|
| Cheapest storage per GB | Slow queries without indexing or partitioning |
| Stores everything — future-proof | "Data swamp" if ungoverned (unknown schema, bad quality) |
| Ideal for ML/data science on raw data | Requires expertise (Spark, Hive, Presto) |
| Flexible schema evolution | ACID transactions need Delta/Iceberg on top |

### Data Warehouse
| Advantage | Disadvantage |
|---|---|
| Fast SQL queries for business analysts | Expensive at scale |
| Strong data governance and quality | Schema changes are expensive |
| Excellent BI tool integration | No raw or unstructured data |
| ACID transactions | Separate system from data lake |

---

## When to use

**Use a Data Lake when:**
- Raw, unstructured, or semi-structured data (logs, clickstreams, JSON events).
- Data science and ML workloads that need full history in native formats.
- Cost-sensitive storage of large volumes (~100TB+).
- You don't yet know all the queries you'll run (exploratory analytics).

**Use a Data Warehouse when:**
- Business analysts run regular SQL reports and dashboards.
- Clean, curated, well-defined metrics (revenue, funnel, retention).
- Fast query SLAs for BI tools.
- Sub-second query performance is required.

**Use a Lakehouse when:**
- You want both ML/data science access to raw data AND fast SQL for analysts.
- You need ACID over a data lake (upserts, deletes for GDPR).
- You want a single storage layer for all analytics.

---

## Common Pitfalls

- **Data swamp**: ingesting data into the lake without schema documentation, data catalog, or quality checks. After 1 year, no one knows what's in the lake. Always maintain a data catalog (AWS Glue, Apache Atlas) and document schemas.
- **Not partitioning data lake files**: Spark reading 1TB of JSON without partitioning scans the entire dataset. Partition by date (`year/month/day`) and high-cardinality filter keys to enable partition pruning.
- **Too many small files**: Spark creates one file per partition task. 10,000 tasks → 10,000 small Parquet files → slow list operations. Use compaction jobs (Delta `OPTIMIZE`) to merge small files regularly.
- **Mixing data lake and warehouse roles without a lakehouse**: running both a full data lake and data warehouse doubles storage costs and requires complex ETL. Consider Delta Lake or Iceberg to unify the two.
- **Not defining an SLA for data freshness**: "the data is in the lake" doesn't tell analysts when they can use it. Define and monitor data latency (ingestion lag) and quality SLOs.

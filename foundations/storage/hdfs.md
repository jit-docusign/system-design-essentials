# HDFS — Hadoop Distributed File System

## What it is

**HDFS** (Hadoop Distributed File System) is a distributed file system designed to store and process **very large datasets** (terabytes to petabytes) reliably across clusters of commodity hardware. It was created as the storage layer of the Hadoop ecosystem and remains foundational to many batch data processing pipelines.

HDFS favors **high throughput over low latency** — it's designed for streaming large sequential reads (MapReduce jobs, Spark batch processing) rather than random-access lookups.

---

## How it works

### Architecture: NameNode and DataNodes

HDFS has a **master-follower** architecture:

```
                      NameNode (master)
                      ┌─────────────────────────────┐
                      │  Metadata store:             │
                      │  - File → list of blocks     │
                      │  - Block → list of DataNodes │
                      │  - Namespace (directory tree)│
                      └───────────┬─────────────────┘
                                  │ heartbeat + block report
              ┌───────────────────┼───────────────────┐
              ▼                   ▼                   ▼
        DataNode 1          DataNode 2          DataNode 3
        ┌────────────┐      ┌────────────┐      ┌────────────┐
        │ block_001  │      │ block_001  │      │ block_002  │
        │ block_002  │      │ block_003  │      │ block_003  │
        │ block_004  │      │ block_004  │      │ block_005  │
        └────────────┘      └────────────┘      └────────────┘
```

**NameNode** (single master):
- Stores the entire file system metadata in memory (fast).
- Does NOT store actual data.
- Tracks: file → list of blocks, block → list of DataNodes holding replicas.
- Journal of all metadata changes persisted to disk (EditLog) + periodic snapshot (FsImage).

**DataNode** (workers, many):
- Store actual data blocks on local disks.
- Send heartbeats to NameNode every 3 seconds (alive signal).
- Send block reports to NameNode every 6 hours (full inventory of stored blocks).
- Serve read requests directly from clients after NameNode provides locations.

### Blocks and Replication

Files are split into large **blocks** (default: 128 MB). Each block is replicated across **N DataNodes** (default replication factor: 3).

```
File: sales_data_2024.csv  (500 MB)

Blocks:
  Block 0: bytes [0, 128MB)      replicated on: N1, N2, N5
  Block 1: bytes [128MB, 256MB)  replicated on: N2, N3, N6
  Block 2: bytes [256MB, 384MB)  replicated on: N1, N4, N6
  Block 3: bytes [384MB, 500MB)  replicated on: N3, N5, N7

If N2 fails: blocks 0 and 1 are now under-replicated (2 replicas)
  → NameNode detects via missing heartbeat
  → schedules re-replication to restore 3 replicas
```

**Rack-awareness**: HDFS places replicas across racks so a single rack failure doesn't lose all replicas. Default policy: 2 replicas in one rack, 1 in another.

### Read Path

```
Client ──► NameNode: "where are the blocks for /data/sales.csv?"
NameNode ──► Client: block 0 → [N1, N2, N5], block 1 → [N2, N3, N6], ...
Client ──► DataNode N1: "read block 0"
DataNode N1 ──► Client: streams 128 MB of data
Client ──► DataNode N2: "read block 1"  (can be parallelized)
...
```

Clients read directly from DataNodes — the NameNode only provides metadata. This allows massive parallel reads across many DataNodes simultaneously.

### Write Path

```
Client ──► NameNode: "create /data/new_file.csv, need 3 replicas per block"
NameNode ──► Client: "write block 0 to N1, N3, N7 (pipeline)"

Client ──► N1 ──► N3 ──► N7  (pipeline: data flows in a chain)
N7 ──► N3 ──► N1 ──► Client  (ACKs flow back)
```

HDFS uses a **write pipeline**: data flows from client → DataNode 1 → DataNode 2 → DataNode 3 in a chain, and ACKs flow back. This saturates a single network path rather than splitting bandwidth to 3 nodes.

### HDFS vs Object Storage (S3)

| Dimension | HDFS | Object Storage (S3) |
|---|---|---|
| Location | On-premise / IaaS cluster | Managed cloud service |
| Compute-storage coupling | Tightly coupled (data locality) | Decoupled (S3 Select / separate compute) |
| Sequential read throughput | Very high (via data locality) | High (multi-threaded GET) |
| Cost model | Hardware + ops team | Per GB/TB + egress |
| Failure handling | NameNode SPOF (mitigated by HA) | Fully managed HA |
| Data locality | Yes (Spark reads local blocks) | No (Spark on EMR reads from S3 over network) |
| Modern trend | Being replaced by S3+Spark/Flink | Primary modern data lake storage |

Modern architectures increasingly use **S3 or GCS** as the data lake storage layer (replacing HDFS), with Spark/Flink as the compute engine. HDFS remains relevant in on-premise Hadoop clusters and environments with high data locality requirements.

### HDFS Federation and High Availability

**NameNode HA** (High Availability): runs an Active + Standby NameNode. Both share an edit log via a **JournalNode quorum** (like a Raft log). Standby can take over in seconds on active failure.

```
Active NameNode ──► JournalNode 1  ──► Standby NameNode (syncs edits)
                ──► JournalNode 2
                ──► JournalNode 3  (quorum write)
```

**HDFS Federation**: multiple independent NameNodes, each managing a portion of the namespace. Enables scaling metadata beyond a single NameNode's memory.

---

## Key Trade-offs

| Advantage | Disadvantage |
|---|---|
| Handles petabyte-scale datasets on commodity hardware | High latency — not suitable for interactive queries |
| High sequential read throughput with data locality | NameNode is a SPOF without HA (memory + single master) |
| Fault-tolerant replication | Large block size creates overhead for many small files |
| Open-source, integrates with Spark, Hive, etc. | Operational complexity: cluster management, rebalancing |
| Batch processing throughput | Being replaced by cloud object storage (S3) for most use cases |

---

## When to use

HDFS remains appropriate for:
- **On-premise big data clusters** where cloud storage is not an option.
- **High data locality requirements**: Spark jobs on HDFS can read from local DataNodes, reducing network I/O for compute-heavy workloads.
- **Legacy Hadoop ecosystems** with existing HDFS pipelines.

Use object storage (S3/GCS) instead of HDFS when:
- Running on cloud infrastructure.
- You want to separate compute and storage scaling.
- You prefer managed services over operating a Hadoop cluster.

---

## Common Pitfalls

- **Small files problem**: HDFS is optimized for large files. Millions of small files (< 128 MB) creates NameNode memory pressure (each file = ~150 bytes of NameNode metadata). Compact small files into Parquet, ORC, or Avro before storing.
- **Single NameNode without HA**: in production, always configure NameNode HA with JournalNodes. A NameNode failure without HA brings down the entire cluster.
- **Not rebalancing**: as DataNodes are added/removed, block distribution becomes skewed. Run the HDFS balancer periodically to maintain even distribution.
- **Treating HDFS like a low-latency store**: HDFS is for batch sequential reads. Running interactive queries or random point lookups directly on HDFS is slow — use Hive with ORC indexes, or better, use a proper OLAP system (ClickHouse, Redshift) for analytics.

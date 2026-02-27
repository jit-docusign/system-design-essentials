# Object Storage

## What it is

**Object storage** is a flat, unstructured storage system that stores data as discrete **objects**, each consisting of:
- **Data**: the binary content (file, image, video, log, backup).
- **Metadata**: custom key-value attributes describing the object.
- **Unique identifier (key)**: a globally unique key used to retrieve the object.

Unlike file systems (hierarchical directories) or block storage (raw disk sectors), object storage has no directory tree. All objects live in a flat namespace (or a simulated hierarchy using key prefixes like `images/2024/01/photo.jpg`).

Object storage excels at **storing and retrieving massive quantities of unstructured data at virtually unlimited scale**. Examples: Amazon S3, Google Cloud Storage, Azure Blob Storage, MinIO.

---

## How it works

### Core Operations

Object storage provides a simple REST API with four core operations:

```
PUT   /bucket/object-key      → upload an object (create or overwrite)
GET   /bucket/object-key      → download an object
DELETE /bucket/object-key     → delete an object
LIST  /bucket?prefix=images/  → list objects with a prefix
```

```python
import boto3

s3 = boto3.client('s3')

# Upload
s3.put_object(
    Bucket='my-bucket',
    Key='images/products/product-001.jpg',
    Body=image_bytes,
    ContentType='image/jpeg',
    Metadata={'product_id': 'prod-001', 'uploaded_by': 'user-123'}
)

# Download
response = s3.get_object(Bucket='my-bucket', Key='images/products/product-001.jpg')
image_bytes = response['Body'].read()

# Generate pre-signed URL (time-limited direct download without credentials)
url = s3.generate_presigned_url(
    'get_object',
    Params={'Bucket': 'my-bucket', 'Key': 'images/products/product-001.jpg'},
    ExpiresIn=3600  # 1 hour
)
```

### Architecture

Object storage is designed for:
- **Horizontal scale-out**: data is spread across thousands of commodity servers.
- **Eventual consistency** (historically): different nodes may briefly return stale data. S3 switched to strong read-after-write consistency in December 2020.
- **High durability**: S3 achieves 11 nines (99.999999999%) by storing replicas across multiple Availability Zones.

```
S3 Architecture (simplified):
  Client ──► S3 Gateway (stateless) ──► Placement Service (metadata)
                                    ──► Storage Nodes (data)
                                         ├── Node 1 (AZ-a): replica 1
                                         ├── Node 2 (AZ-b): replica 2
                                         └── Node 3 (AZ-c): replica 3
```

### Storage Classes

Object storage offers tiers based on access frequency and cost:

| Tier | Retrieval Time | Cost | Use Case |
|---|---|---|---|
| **Standard** | Immediate | Highest | Frequently accessed data |
| **Infrequent Access (IA)** | Immediate | Lower storage, retrieval fee | Monthly access patterns |
| **Intelligent Tiering** | Immediate | Auto-tiers based on access | Unknown access patterns |
| **Glacier Instant** | Milliseconds | Very low | Archives, quarterly access |
| **Glacier Deep Archive** | Hours | Lowest | 7-year compliance archives |

Lifecycle policies automatically transition objects between tiers:
```json
{
  "Rules": [{
    "Status": "Enabled",
    "Transitions": [
      {"Days": 30, "StorageClass": "STANDARD_IA"},
      {"Days": 90, "StorageClass": "GLACIER"},
      {"Days": 365, "StorageClass": "DEEP_ARCHIVE"}
    ],
    "Expiration": {"Days": 2555}  // delete after 7 years
  }]
}
```

### Typical Use Cases

```
User-generated content:
  User uploads profile photo ──► stored in S3
  CDN (CloudFront/Fastly) ──► serves from edge cache ──► users worldwide

Static website hosting:
  HTML/CSS/JS ──► S3 bucket ──► served via CDN

Data lake:
  ETL pipeline ──► raw events in S3 (Parquet/JSON)
  Athena/Spark ──► query directly from S3

Backup and disaster recovery:
  Database dumps, snapshots ──► S3 with versioning enabled
  Glacier Deep Archive ──► 7-year compliance retention

Application state and configuration:
  Large model weights, config bundles ──► S3 ──► loaded on startup
```

### Object Versioning

With versioning enabled, every PUT creates a new version of the object. Deletes create a "delete marker" but don't erase old versions.

```
Bucket: my-bucket  (versioning enabled)
Key: config.json

Version history:
  v1 (2024-01-01): { "max_connections": 100 }
  v2 (2024-02-01): { "max_connections": 200 }
  v3 (2024-03-01): DELETE MARKER
  
GET config.json (current) → 404 (delete marker)
GET config.json?versionId=v2 → returns v2 content
```

---

## Key Trade-offs

| Advantage | Disadvantage |
|---|---|
| Virtually unlimited scale | High latency vs local file system (tens of ms per operation) |
| 11-nines durability with AZ replication | Not suitable for high-IOPS workloads (databases) |
| Low cost at scale ($0.023/GB/month typical) | Eventual consistency in multi-region settings |
| Simple REST API | No append operations — must rewrite whole object |
| Lifecycle policies for cost management | Directory operations are simulated (list is expensive) |
| Built-in versioning and access control | Per-operation API costs add up at high request rates |

---

## When to use

- **User-generated media**: profile photos, videos, documents.
- **Static website assets** and frontend bundles.
- **Data lake storage**: raw events, logs, Parquet files for analytics.
- **Backup and archival**: database backups, DR snapshots.
- **Software distribution**: binary releases, ML model weights, large dependencies.

Object storage is the wrong choice for:
- Databases that need random read/write to fixed sectors → use block storage.
- Shared file systems where multiple VMs mount the same directory → use file storage (EFS/NFS).
- Caching hot data → use Redis/Memcached.

---

## Common Pitfalls

- **Storing many small files**: GetObject has per-request latency (~10–50ms). Millions of 1KB files are painful. Combine small files into larger Parquet/ORC files for analytics; use a cache for hot small objects.
- **Public S3 buckets**: making a bucket or objects public by default is a frequent security incident. Enable "Block Public Access" at the account level and use pre-signed URLs or CDN for serving content.
- **No lifecycle policies**: without lifecycle rules, objects accumulate indefinitely. Always configure policies to transition cold data to cheaper tiers and delete expired data.
- **Using S3 as a file system**: frequent list operations and renames (in S3 = copy + delete) at scale are expensive and slow. Use a proper file system (HDFS, EFS) for workflows requiring file semantics.

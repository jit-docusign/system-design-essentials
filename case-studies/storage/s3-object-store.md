# Design an Object Store (S3 / GCS Internals)

## Problem Statement

Design a highly durable, highly available object storage service. Users should be able to store arbitrarily-sized blobs (photos, videos, backups, logs) identified by a (bucket, key) pair, with 11 nines (99.999999999%) durability and 99.99% availability. This models Amazon S3 internals.

**Scale requirements**:
- 1 trillion objects stored
- 1 exabyte (1 million TB) of total data
- 500,000 PUT requests per second
- 2 million GET requests per second
- Objects from 1 byte to 5 TB
- Durability: 99.999999999% (data loss is 1 object per 100 billion per year)

---

## Architecture Overview

```
┌────────────────────────────────────────────────────────────────────┐
│  Client Layer                                                      │
│  PUT s3://my-bucket/photos/cat.jpg [5 MB]                         │
│  GET s3://my-bucket/photos/cat.jpg                                │
└─────────────────────────────────┬──────────────────────────────────┘
                                  │  HTTP REST
                                  ▼
                        ┌─────────────────┐
                        │  Frontend Nodes  │
                        │  (API Gateway)   │
                        │  - Auth (IAM)    │
                        │  - Rate limiting │
                        │  - Route request │
                        └────────┬────────┘
                                 │
              ┌──────────────────┼──────────────────┐
              │                  │                  │
              ▼                  ▼                  ▼
     ┌────────────────┐  ┌──────────────┐  ┌──────────────────┐
     │ Metadata Store │  │ Data Store   │  │ Data Store       │
     │                │  │ Node Cluster │  │ Node Cluster     │
     │ (bucket+key →  │  │ (actual      │  │ (replicas)       │
     │  location)     │  │  bytes)      │  │                  │
     └────────────────┘  └──────────────┘  └──────────────────┘
```

---

## How It Works

### 1. Object Placement — Consistent Hashing

Objects are striped across storage nodes using consistent hashing on the object key:

```python
import hashlib
import bisect

class ConsistentHash:
    def __init__(self, nodes: list[str], replicas=150):
        self.replicas = replicas
        self.ring = {}       # hash_position → node
        self.sorted_keys = []
        
        for node in nodes:
            self.add_node(node)
    
    def add_node(self, node: str):
        for i in range(self.replicas):
            key = self._hash(f"{node}:{i}")
            self.ring[key] = node
            bisect.insort(self.sorted_keys, key)
    
    def remove_node(self, node: str):
        for i in range(self.replicas):
            key = self._hash(f"{node}:{i}")
            self.ring.pop(key, None)
            idx = bisect.bisect_left(self.sorted_keys, key)
            if idx < len(self.sorted_keys) and self.sorted_keys[idx] == key:
                self.sorted_keys.pop(idx)
    
    def get_node(self, object_key: str) -> str:
        if not self.ring:
            return None
        hash_val = self._hash(object_key)
        idx = bisect.bisect_right(self.sorted_keys, hash_val) % len(self.sorted_keys)
        return self.ring[self.sorted_keys[idx]]
    
    def get_nodes(self, object_key: str, count: int) -> list[str]:
        """Get 'count' distinct nodes for replication"""
        if not self.ring:
            return []
        hash_val = self._hash(object_key)
        idx = bisect.bisect_right(self.sorted_keys, hash_val) % len(self.sorted_keys)
        
        nodes = []
        seen = set()
        for i in range(len(self.sorted_keys)):
            pos = (idx + i) % len(self.sorted_keys)
            node = self.ring[self.sorted_keys[pos]]
            if node not in seen:
                nodes.append(node)
                seen.add(node)
            if len(nodes) == count:
                break
        return nodes
    
    def _hash(self, key: str) -> int:
        return int(hashlib.sha256(key.encode()).hexdigest()[:8], 16)

# Usage
nodes = [f"storage-node-{i}" for i in range(100)]
ring = ConsistentHash(nodes)

# Object "photos/cat.jpg" → 3 replicas on different nodes
primary, replica1, replica2 = ring.get_nodes("photos/cat.jpg", count=3)
# → ["storage-node-47", "storage-node-12", "storage-node-83"]
```

### 2. Durability via Erasure Coding

Storing 3 full replicas costs 3× storage. Erasure coding achieves similar (better) durability at lower overhead:

```
Reed-Solomon (12, 4) erasure coding:
  - Split object into 12 data shards
  - Compute 4 parity shards (mathematically derived from data shards)
  - Store all 16 shards on 16 different nodes (ideally different AZs)
  - Can reconstruct full object from ANY 12 of 16 shards
  - Can tolerate 4 simultaneous node failures
  - Storage overhead: 16/12 = 1.33× vs 3× for full replication

Example: 120 MB object
  → 12 data shards × 10 MB = 120 MB data
  → 4 parity shards × 10 MB = 40 MB parity
  → 160 MB total stored (vs 360 MB for 3× replication)

Durability math:
  If each node has 0.001% failure probability per year:
  Losing all 12+ nodes simultaneously: astronomically unlikely
  S3 uses 14+6 = 20 shards across 3+ AZs → 11 nines durability
```

```python
# Conceptual erasure coding (simplified)
import galois

def encode_erasure(data: bytes, k=12, m=4) -> list[bytes]:
    """
    Encode data into k data shards + m parity shards.
    Uses GF(2^8) arithmetic for Reed-Solomon.
    k = data shards, m = parity shards
    """
    # Split data into k equal shards
    shard_size = (len(data) + k - 1) // k
    padded = data.ljust(shard_size * k, b'\x00')
    data_shards = [padded[i*shard_size:(i+1)*shard_size] for i in range(k)]
    
    # Compute parity shards via Reed-Solomon over GF(2^8)
    # (In practice: use a library like liberasurecode or PyECLib)
    gf = galois.GF(2**8)
    # ... (parity computation with Vandermonde matrix)
    
    return data_shards  # + parity_shards in real implementation

def decode_erasure(available_shards: dict[int, bytes], k=12, m=4) -> bytes:
    """
    Reconstruct data from any k-of-(k+m) available shards.
    available_shards: {shard_index: shard_data}
    """
    if len(available_shards) < k:
        raise ValueError(f"Need at least {k} shards, got {len(available_shards)}")
    # Use inverse of generator matrix to recover missing shards
    # ... (matrix inversion in GF(2^8))
```

### 3. Metadata Store (The Hard Part)

1 trillion objects requires a distributed metadata service:

```
Metadata for each object:
  - bucket: "my-bucket"
  - key: "photos/cat.jpg"
  - version_id: "abc123"
  - size: 5_242_880  (bytes)
  - etag: "abc123def456"  (MD5 hash for integrity verification)
  - content_type: "image/jpeg"
  - storage_class: "STANDARD" | "INFREQUENT_ACCESS" | "GLACIER"
  - created_at: 1704067200
  - shard_locations: [{"node": "node-47", "shard": 0}, ...]  (for erasure coded)
  - user_metadata: {"author": "Alice"}

Storage: 
  Option A: Distributed KV Store (e.g., RocksDB on each node with consistent hashing)
  Option B: CockroachDB / Spanner (distributed SQL for ACID guarantees)
  
  S3 uses a custom distributed metadata service (similar to Dynamo) with eventual
  consistency for bucket listings, strong consistency for object GET-after-PUT.

Schema (conceptual):
  Key:   {bucket}{key}{version_id}  (lexicographic ordering for list operations)
  Value: protobuf blob of all metadata fields
```

### 4. PUT Object (Write Path)

```
Client: PUT /my-bucket/photos/cat.jpg
        Content-Length: 5242880
        Body: <binary data>

1. Frontend node:
   a. Authenticate request (IAM policy check)
   b. Validate bucket exists, client has write permission
   c. Generate version_id = random 16-byte hex
   d. Stream body to erasure encoding pipeline

2. Data pipeline:
   a. Compute MD5 checksum on-the-fly (for ETag)
   b. Apply encryption if bucket has SSE enabled (AES-256)
   c. Erasure encode: split into 12 data shards + 4 parity shards

3. Write to storage nodes (parallel, fan-out):
   - Send each shard to its respective storage node
   - Wait for acknowledgment from k+1=13 nodes (quorum write)
   - If a node is slow/down: skip it (will backfill via repair job)
   Status: write succeeds as long as 13 of 16 nodes respond in time

4. Write metadata:
   - metadata_store.put(bucket, key, version_id, shard_locations, etag, size, ...)

5. Return 200 OK with ETag to client
```

### 5. GET Object (Read Path)

```python
def get_object(bucket: str, key: str, version_id: str = None) -> bytes:
    # 1. Lookup metadata
    meta = metadata_store.get(bucket, key, version_id or "latest")
    if not meta:
        raise KeyError("NoSuchKey")
    
    shard_locations = meta["shard_locations"]  # list of {node, shard_index}
    
    # 2. Issue parallel GET requests to all 16 nodes
    #    But we only need 12 to succeed
    import asyncio
    
    async def fetch_shard(node: str, shard_index: int):
        try:
            data = await storage_client.get(node, f"{meta['object_id']}/{shard_index}")
            return shard_index, data
        except Exception:
            return shard_index, None  # node unavailable
    
    tasks = [fetch_shard(sl["node"], sl["shard_index"]) for sl in shard_locations]
    results = asyncio.run(asyncio.gather(*tasks))
    
    # 3. Collect first 12 successful shards
    available = {idx: data for idx, data in results if data is not None}
    
    if len(available) < 12:
        raise IOError("Not enough shards available for reconstruction")
    
    # 4. Erasure decode
    return decode_erasure(available, k=12, m=4)
```

### 6. Multipart Upload for Large Objects

Objects up to 5 TB need multipart upload:

```
1. Initiate: POST /my-bucket/video.mp4?uploads
   → Returns: UploadId = "xyz789"

2. Upload parts (can be parallel):
   PUT /my-bucket/video.mp4?partNumber=1&uploadId=xyz789  → ETag: "part1-etag"
   PUT /my-bucket/video.mp4?partNumber=2&uploadId=xyz789  → ETag: "part2-etag"
   PUT /my-bucket/video.mp4?partNumber=3&uploadId=xyz789  → ETag: "part3-etag"
   (each part ≥ 5 MB, except last part)
   (parts uploaded in any order, from multiple machines in parallel)

3. Complete: POST /my-bucket/video.mp4?uploadId=xyz789
   Body: [{"PartNumber": 1, "ETag": "part1-etag"}, ...]
   → Server assembles parts, stores final object
   → Returns final ETag (MD5 of part ETags)

4. Abort (if failed): DELETE /my-bucket/video.mp4?uploadId=xyz789
   → Cleans up all partial uploads (garbage collected after 7 days if not completed)
```

### 7. Pre-signed URLs

Allow clients to upload/download directly without sending credentials:

```python
import boto3
from datetime import timedelta

s3 = boto3.client("s3", region_name="us-east-1")

# Generate presigned PUT URL (expires in 15 minutes)
upload_url = s3.generate_presigned_url(
    ClientMethod="put_object",
    Params={
        "Bucket": "my-bucket",
        "Key": "uploads/user-123/avatar.jpg",
        "ContentType": "image/jpeg",
        "ContentLength": 512000
    },
    ExpiresIn=900  # 15 minutes
)

# Client uploads directly to S3 (no server needed in loop)
# curl -X PUT "{upload_url}" -H "Content-Type: image/jpeg" --data-binary @avatar.jpg

# Generate presigned GET URL (expires in 1 hour)
download_url = s3.generate_presigned_url(
    ClientMethod="get_object",
    Params={"Bucket": "my-bucket", "Key": "uploads/user-123/avatar.jpg"},
    ExpiresIn=3600
)
```

**How pre-signing works internally**:
```
URL = base_url + ?X-Amz-Signature={HMAC-SHA256(secret_key, canonical_request)}
The signature encodes: who, what bucket/key, expiry time, allowed HTTP method.
S3 validates the signature on each request — no auth header needed from client.
```

---

## Key Design Decisions

| Decision | Option A | Option B | S3 Choice |
|---|---|---|---|
| Durability model | 3× replication (3× storage) | Erasure coding 12+4 (1.33× storage) | Erasure coding (saves ~55% storage vs 3×) |
| Metadata consistency | Eventual consistent (DynamoDB-style) | Strong consistent | Strong consistency since Dec 2020 |
| Bucket operations | Flat namespace only | Simulated folders (key prefix) | Key prefix (no real folders; "/" is just a delimiter) |
| Object versioning | Single version | Full version history | Configurable (versioning off by default) |
| Storage tiers | Single tier | Hot/warm/cold/archive | Four tiers: S3 Standard, IA, Glacier, Glacier Deep Archive |

---

## Scaling Challenges

| Challenge | Problem | Solution |
|---|---|---|
| Hot bucket / hot key | Popular prefix like "images/" → hot partition | Use object key prefixes like "{hash}/" to distribute load |
| Large LIST operations | LIST bucket with 1B objects → slow scan | Maintain separate index of keys per bucket; paginate with continuation tokens |
| Tombstone accumulation | Deleted objects leave tombstones → scan overhead | Compaction background job removes tombstones after replication guaranteed |
| Cross-region replication | Disaster recovery in another region | Async CRR (Cross-Region Replication): stream changes to replica region via SQS/event log |
| Bit rot | Silent disk corruption over months | Continuous integrity scrubbing: read every shard, verify checksum, repair if corrupted |

---

## Common Interview Mistakes

1. **Storing objects in a database**: Object storage is NOT a relational DB. Files go in a distributed block store; only metadata goes in a DB/KV store.

2. **Not distinguishing metadata from data**: The metadata store (bucket+key → location) and the data store (actual bytes) are separate systems with different scaling properties.

3. **Forgetting erasure coding**: Saying "store 3 copies" is fine for durability but shows you don't know about the storage efficiency of erasure coding that real object stores use.

4. **Ignoring pre-signed URLs**: Applications should not proxy file uploads through their servers. Pre-signed URLs allow direct client-to-S3 uploads — major for scalability.

5. **Assuming folders exist**: S3 has no real folders. `s3://bucket/a/b/c.jpg` is a flat key with "/" delimiters. `LIST` with prefix "a/b/" simulates directory listing.

6. **Not addressing multipart upload**: Files up to 5 TB require multipart. Forgetting this means your design only handles small objects.

7. **Confusing consistency model**: S3 was eventually consistent for list operations until 2020. NOW it is strongly consistent (GET-after-PUT is guaranteed). Know this has changed.

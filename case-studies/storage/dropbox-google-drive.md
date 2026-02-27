# Design a Cloud File Storage (Dropbox / Google Drive)

## Problem Statement

Design a cloud file storage and synchronization service that allows users to upload files from multiple devices, have them synced in near-real-time, share with collaborators, and access files offline. Similar to Dropbox, Google Drive, iCloud Drive.

**Scale requirements**:
- 500 million users; 50 million daily actives
- 200 PB total storage
- 10 billion files stored
- Average file size: 500 KB; max file size: 5 GB
- 1 million file upload/updates per minute
- < 2 second sync propagation to other devices

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│  Client (Desktop Sync App / Web / Mobile)                           │
│  - Watch filesystem for changes                                     │
│  - Chunk files, compute content hashes                              │
│  - Upload only changed chunks (delta sync)                          │
└───────────────┬─────────────────────────────────────────────────────┘
                │
                │  HTTPS (chunk upload / metadata API)
                ▼
        ┌───────────────┐
        │  API Gateway  │
        └───────┬───────┘
                │
     ┌──────────┼───────────────────┐
     │          │                   │
     ▼          ▼                   ▼
 Metadata    Upload Service      Sync/Notification
 Service     (Chunk Store)       Service (WebSocket)
     │          │                   │
     │          │                   │
     ▼          ▼                   ▼
PostgreSQL   Object Store        WebSocket Server
(file tree,  (S3 / GCS)          + Redis Pub/Sub
 versions,    Block Layer         (push change events
 shares)      (dedup by hash)      to connected clients)
```

---

## How Chunked Upload Works

### Chunk Strategy

Split large files into 4 MB chunks. Each chunk is identified by its SHA-256 content hash:

```python
import hashlib
import os

CHUNK_SIZE = 4 * 1024 * 1024  # 4 MB

def chunk_file(file_path: str) -> list[dict]:
    chunks = []
    with open(file_path, "rb") as f:
        chunk_index = 0
        while True:
            data = f.read(CHUNK_SIZE)
            if not data:
                break
            chunk_hash = hashlib.sha256(data).hexdigest()
            chunks.append({
                "index": chunk_index,
                "hash": chunk_hash,
                "size": len(data),
                "data": data  # only sent if not already on server
            })
            chunk_index += 1
    return chunks

def compute_file_hash(chunks: list[dict]) -> str:
    """File identity = hash of all chunk hashes (Merkle-like root)"""
    combined = b"".join(c["hash"].encode() for c in chunks)
    return hashlib.sha256(combined).hexdigest()
```

### Upload Flow

```
Client                         Upload Service                  Object Store
  │                                  │                              │
  │  1. check_chunks([hash1, hash2,  │                              │
  │     hash3, hash4])               │                              │
  │─────────────────────────────────>│                              │
  │                                  │  check which hashes exist    │
  │                                  │─────────────────────────────>│
  │  2. need_upload=[hash1, hash3]   │  (hash → bool from index)    │
  │<──────────────────────────────── │                              │
  │                                  │                              │
  │  3. upload chunks 1 and 3 only   │                              │
  │─────────────────────────────────>│                              │
  │     (skip hash2, hash4: exists)  │  store chunk by content hash │
  │                                  │─────────────────────────────>│
  │  4. commit_file(metadata)        │                              │
  │─────────────────────────────────>│                              │
  │                                  │  write metadata to DB        │
  │  5. ACK: file version v3         │                              │
  │<──────────────────────────────── │                              │
```

**Key insight**: If chunk hash already exists on the server (uploaded by this user or anyone else), skip uploading it. This achieves:
- **Delta sync**: 100 MB file, user edits 1 page → only 1 chunk (4 MB) uploaded
- **Cross-user deduplication**: Two users upload the same video → stored once (saves ~40% storage on average)

### Deduplication at the Block Level

```python
# Block/Chunk Store schema (conceptual)
#
# block_store table:
#   hash    VARCHAR(64) PRIMARY KEY  -- SHA-256 of chunk content
#   size    INT
#   ref_count INT                   -- how many files reference this chunk
#   stored_at OBJECT_STORE_PATH     -- e.g., s3://blocks/ab/cd/abcd1234...
#
# file_blocks table (file → ordered list of chunks):
#   file_id   BIGINT
#   version   INT
#   position  INT
#   chunk_hash VARCHAR(64) REFERENCES block_store(hash)
#   PRIMARY KEY (file_id, version, position)

# Upload a chunk:
def upload_chunk(chunk_data: bytes) -> str:
    chunk_hash = sha256(chunk_data)
    # Idempotent: if exists, just increment ref_count
    db.execute("""
        INSERT INTO block_store (hash, size, ref_count)
        VALUES (%s, %s, 1)
        ON CONFLICT (hash) DO UPDATE SET ref_count = ref_count + 1
    """, [chunk_hash, len(chunk_data)])
    
    # Only write to object store if new
    if not object_store.exists(chunk_hash):
        object_store.put(f"blocks/{chunk_hash[:2]}/{chunk_hash[2:4]}/{chunk_hash}", chunk_data)
    
    return chunk_hash
```

---

## Metadata Service

Stores the file tree, versions, and sharing permissions in a relational DB:

```sql
-- Files / folders hierarchy
CREATE TABLE nodes (
    id          BIGSERIAL PRIMARY KEY,
    owner_id    BIGINT NOT NULL,
    parent_id   BIGINT REFERENCES nodes(id),
    name        TEXT NOT NULL,
    type        VARCHAR(10) NOT NULL,  -- 'file' | 'folder'
    created_at  TIMESTAMPTZ DEFAULT NOW(),
    modified_at TIMESTAMPTZ DEFAULT NOW(),
    is_deleted  BOOLEAN DEFAULT FALSE,  -- soft delete
    UNIQUE (parent_id, name)  -- no two files with same name in same folder
);

-- File versions (immutable history)
CREATE TABLE file_versions (
    id          BIGSERIAL PRIMARY KEY,
    node_id     BIGINT REFERENCES nodes(id),
    version     INT NOT NULL,
    size_bytes  BIGINT NOT NULL,
    content_hash VARCHAR(64) NOT NULL,   -- full file hash (for quick equality check)
    created_at  TIMESTAMPTZ DEFAULT NOW(),
    created_by  BIGINT NOT NULL,
    chunk_count INT NOT NULL,
    PRIMARY KEY (node_id, version)
);

-- File chunks in order
CREATE TABLE file_chunks (
    node_id     BIGINT,
    version     INT,
    position    INT,
    chunk_hash  VARCHAR(64) NOT NULL,
    PRIMARY KEY (node_id, version, position),
    FOREIGN KEY (node_id, version) REFERENCES file_versions(node_id, version)
);

-- Sharing
CREATE TABLE shares (
    id          BIGSERIAL PRIMARY KEY,
    node_id     BIGINT REFERENCES nodes(id),   -- file or folder being shared
    shared_with BIGINT,                         -- NULL if public link
    share_link  VARCHAR(32) UNIQUE,             -- random token for public share
    permission  VARCHAR(10),                    -- 'view' | 'edit' | 'owner'
    expires_at  TIMESTAMPTZ
);
```

---

## Sync Protocol

### Change Detection (Client Side)

```
On startup and on filesystem events (inotify on Linux, FSEvents on macOS):

1. File created/modified:
   - Chunk the file; compute hashes
   - Compare with last known state (stored in local .dropbox SQLite cache)
   - If hashes differ → initiate upload (only new/changed chunks)

2. File renamed/moved:
   - Metadata-only update (no chunk re-upload)
   - PUT /files/{id}/move {new_parent_id, new_name}

3. File deleted:
   - Soft delete: file moves to trash, recoverable for 30 days
   - DELETE /files/{id}

Local state cache (SQLite, per device):
  node_id | path | version | content_hash | sync_status
  --------+------+---------+--------------+------------
   42     | /docs/budget.xlsx | v5 | abc123 | synced
   43     | /photos/trip.jpg  | v1 | def456 | syncing
```

### Conflict Resolution

When two clients modify the same file offline:

```
Device A (offline for 2 hours): budget.xlsx → version A
Device B (online, synced):       budget.xlsx → version B (on server)

When Device A reconnects:
  Server has v_server, Device A has v_local
  
  Case 1: content_hash matches → no conflict, skip
  Case 2: v_local base != v_server base (concurrent edits):
    → Create conflict copy:
       "budget.xlsx" stays as server version (wins)
       "budget (Device A's conflicted copy 2024-01-15).xlsx" = local version
    → Notify both users

No automatic 3-way merge for binary files (Excel, images).
Google Docs handles merge via operational transforms (real-time collaboration),
but for arbitrary file types, Dropbox uses last-writer-wins + copy-on-conflict.
```

### Real-Time Sync via Long Poll / WebSocket

```python
# On file commit, publish change event to Redis Pub/Sub
def commit_file(user_id: int, node_id: int, version: int):
    # Save to DB
    save_file_version(node_id, version)
    
    # Notify all devices owned by this user
    event = {
        "type": "file_updated",
        "node_id": node_id,
        "version": version,
        "user_id": user_id
    }
    redis.publish(f"user:{user_id}:changes", json.dumps(event))
    
    # Also notify collaborators with access to this file
    collaborators = get_file_collaborators(node_id)
    for collab_id in collaborators:
        redis.publish(f"user:{collab_id}:changes", json.dumps(event))

# WebSocket server subscribes per connected user
class SyncWebSocketHandler:
    async def on_connect(self, user_id: int):
        # Subscribe to this user's change channel
        async with redis.subscribe(f"user:{user_id}:changes") as channel:
            async for message in channel:
                await self.send(message)  # push to device
```

---

## Large File Upload (Multipart / Resumable)

For files > 100 MB, use resumable uploads so a network interruption doesn't require restarting:

```
1. Client: POST /uploads/initiate
   → Server returns upload_id

2. Client: for each 4 MB chunk:
   PUT /uploads/{upload_id}/chunks/{index}
   Content-Range: bytes 0-4194303/104857600
   Body: <chunk data>
   
   Server: store chunk in temp location, track which chunks received

3. Client: if upload interrupted, query server:
   GET /uploads/{upload_id}/status
   → {received_chunks: [0,1,2,4,5], missing: [3]}
   
   Client resumes from chunk 3 only.

4. Client: POST /uploads/{upload_id}/complete
   → Server assembles chunks, moves to permanent store
   → Deduplication check on each chunk
   → Commit metadata to DB
```

---

## Key Design Decisions

| Decision | Option A | Option B | Usually chosen |
|---|---|---|---|
| Chunk size | 4 MB | Variable (rsync algorithm) | 4 MB fixed (simple, good average) |
| Deduplication | Per-user only | Cross-user (global dedup) | Cross-user (40% storage savings) |
| Conflict resolution | Last-writer-wins + copy | 3-way merge | Last-writer-wins + copy (binary files) |
| Metadata DB | PostgreSQL | Cassandra | PostgreSQL (strong consistency for file tree) |
| Sync protocol | WebSocket push | Long polling | WebSocket (lower latency) |
| File versioning | Keep all versions | Keep N days/N versions | 30-day trash + unlimited versions for paid |

---

## Scaling Challenges

| Challenge | Problem | Solution |
|---|---|---|
| Metadata DB hot users | Celebrity folder shared with 10M users → metadata hot row | Shard metadata by user_id; cache popular share metadata in Redis |
| Block store hot chunks | Same popular file downloaded by millions | CDN in front of block store; presigned URLs with short TTL |
| Namespace collisions | concurrent renames in same folder | Optimistic locking: version check on folder node before commit |
| Storage cost | 200 PB of raw data | Cold files (no access > 90 days) → Glacier; compression at block level |
| Sync storms on reconnect | Thousands of devices reconnect after server outage → flood of sync requests | Exponential backoff on client reconnect; jitter to spread sync requests |

---

## Hard-Learned Engineering Lessons

1. **Not chunking files**: designing a system that uploads entire files on every change misses the fundamental delta-sync insight of Dropbox.

2. **Missing deduplication**: the same 100 MB video uploaded by 1,000 users should be stored once. Block-level dedup by content hash is a core storage optimization.

3. **Ignoring conflict resolution**: concurrent edits are the hardest part of sync. "Last writer wins" is acceptable but must be stated explicitly — any real sync system must have a clear, documented conflict policy.

4. **Using a single metadata DB without sharding**: 10 billion files in a single Postgres instance won't work. Shard by owner_id or use a distributed metadata service.

5. **Forgetting offline support**: describe how the client tracks local state (SQLite on device), so it can identify deltas on reconnect.

6. **Not discussing resumable uploads**: a 5 GB file upload that fails at 4.9 GB must not restart from zero. Chunked + resumable uploads are required for large files.

7. **Conflating file storage with Google Docs**: Google Drive for file sync ≠ Google Docs collaborative editing. Docs uses operational transforms; Drive uses conflict copies. Know the difference.

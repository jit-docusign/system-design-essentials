# File Storage (Network File Systems)

## What it is

**File storage** (network-attached storage) presents data as a traditional file system — directories, subdirectories, and files — accessible over the network via standard protocols like **NFS** (Network File System) or **SMB/CIFS**. Multiple servers can mount the same file system simultaneously and access a shared namespace.

Unlike block storage (which requires a single VM to own the disk) or object storage (which requires an HTTP API), file storage gives applications a familiar POSIX interface (`open()`, `read()`, `write()`, `mkdir()`) over a network mount. Examples: Amazon EFS, Google Filestore, Azure Files, on-premises NetApp/NFS.

---

## How it works

### Mount and Access Model

```
                   NFS Server / EFS
                   ┌─────────────────────────┐
                   │  /shared                 │
                   │    ├── /uploads          │
                   │    │     ├── img001.jpg  │
                   │    │     └── img002.jpg  │
                   │    └── /configs          │
                   └──────────┬──────────────┘
                              │ NFS protocol (TCP/UDP 2049)
         ┌────────────────────┼────────────────────┐
         ▼                    ▼                    ▼
       VM-1                  VM-2                 VM-3
  (mount: /mnt/shared)  (mount: /mnt/shared)  (mount: /mnt/shared)
  
All three VMs see the same files; writes by VM-1 are visible to VM-2 and VM-3.
```

```bash
# Mount an NFS share on Linux
sudo mount -t nfs4 fs-0abc123.efs.us-east-1.amazonaws.com:/ /mnt/efs

# Or via /etc/fstab for persistent mounts:
# fs-0abc123.efs.us-east-1.amazonaws.com:/ /mnt/efs nfs4 defaults,_netdev 0 0

# Now use normally:
ls /mnt/efs/uploads/
cp local-file.txt /mnt/efs/uploads/new-file.txt
```

### NFS Protocol

NFS is the dominant file storage protocol. Key properties:

**NFS v4 (current standard)**:
- TCP only (reliable delivery).
- Stateful protocol with advanced locking.
- Supports ACLs and Kerberos authentication.

**Key NFS operations**:
```
LOOKUP   → resolve a path to a file handle
READ     → read bytes from a file
WRITE    → write bytes to a file
OPEN     → obtain a stateful open/lock handle
LOCK     → advisory or mandatory byte-range locks
COMMIT   → flush writes to stable storage (fsync)
```

**Client-side caching**: NFS clients cache file data locally for performance. This introduces **cache coherency** challenges — VM-1's cached view may be stale if VM-2 modified the file. NFS v4 uses **delegations** to allow exclusive cached access when safe.

### Amazon EFS (Elastic File System)

EFS is a managed NFS service that scales automatically, with no provisioning of capacity.

```
EFS characteristics:
  - Petabyte-scale: grows and shrinks automatically as files are added/removed
  - Multi-AZ: data distributed across 3+ AZs
  - Latency: ~1–3ms per operation (slightly higher than EBS for single-file access)
  - Performance modes:
      General Purpose: low latency, recommended for most workloads
      Max I/O: higher throughput, higher latency (for massively parallel workloads)
  - Throughput modes:
      Bursting: throughput scales with storage size (50 MB/s per TB)
      Provisioned: independent IOPS and throughput specification
      Elastic: auto-scales based on actual workload
```

```python
# Python writing to EFS via the mounted path — no SDK needed
import os

efs_path = '/mnt/efs/uploads'
os.makedirs(efs_path, exist_ok=True)

with open(os.path.join(efs_path, 'report-2024-01.pdf'), 'wb') as f:
    f.write(pdf_bytes)
```

### Common Use Cases

**Shared application assets**: multiple web server instances access the same user-uploaded files:
```
Upload Server ──► /mnt/efs/uploads/user-123/photo.jpg  (write)
Web Server 1  ──► /mnt/efs/uploads/user-123/photo.jpg  (read)
Web Server 2  ──► /mnt/efs/uploads/user-123/photo.jpg  (read)
```

**Container persistent volumes**: Kubernetes pods on different nodes share a ReadWriteMany PersistentVolumeClaim:
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-files
spec:
  accessModes:
  - ReadWriteMany       # multiple pods can mount simultaneously
  storageClassName: efs-sc
  resources:
    requests:
      storage: 100Gi
```

**Content management**: CMS platforms (WordPress, Drupal) that store uploaded media on a shared file system.

**Machine learning**: training datasets that multiple GPU workers read from simultaneously.

**Lift-and-shift migration**: legacy applications hardcoded to read/write local file paths can use NFS mounts to gain network storage with minimal code changes.

### When File Storage is Not the Right Choice

| Use Case | Right Storage |
|---|---|
| Database data files (high random IOPS) | Block storage (EBS) |
| Static website assets / CDN | Object storage (S3) |
| Archival / backup | Object storage (S3 Glacier) |
| Caching hot data | Redis / Memcached |
| Single-VM high-performance workload | Block storage (EBS) |

---

## Key Trade-offs

| Advantage | Disadvantage |
|---|---|
| Shared access across multiple VMs/pods | Higher latency than local block storage |
| Familiar POSIX file system interface | Not globally accessible (region/VPC-bound) |
| No capacity provisioning needed (EFS) | More expensive than object storage for equivalent data |
| Easy lift-and-shift for legacy applications | Locking can cause contention under heavy concurrent writes |
| Multi-AZ durability (EFS) | Not suitable for very high IOPS database workloads |

---

## Common Pitfalls

- **Using NFS for a database**: databases rely on low-latency random read/write with strict fsync guarantees. NFS introduces latency and inconsistency between client-side cache and server. Use block storage for all database data files.
- **Not setting NFS mount options**: default NFS mount options may not be optimal. For latency-sensitive workloads, tune `rsize`, `wsize` (read/write buffer size), and `async` vs `sync` options.
- **Ignoring NFS latency**: each file operation crosses the network (1–3ms). An application that makes thousands of small sequential file operations per request will see significant overhead. Batch operations or cache locally.
- **Uncapped EFS burst credits**: EFS Bursting mode has a burst credit system. If you exhaust credits (common after large data migrations), throughput drops to baseline. Switch to Provisioned or Elastic throughput mode for consistent workloads.

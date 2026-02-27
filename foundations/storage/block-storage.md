# Block Storage

## What it is

**Block storage** divides data into fixed-size **blocks** (e.g., 4KB, 8KB) and stores each block independently, identified by a block address. The storage system presents these blocks as a raw disk to the operating system, which then formats the disk with a file system (ext4, NTFS, XFS) and manages files on top.

Block storage is the storage type underlying virtual machine disks, databases, and anything requiring **low-latency random read/write access** to specific data locations. Examples: Amazon EBS (Elastic Block Store), Google Persistent Disk, Azure Managed Disks.

---

## How it works

### Core Concept

```
Application
    │
    ▼
File System (ext4/XFS/NTFS)  ←── manages files, directories, inodes
    │
    ▼
Volume Driver / Block Device (/dev/sda1)  ←── reads/writes blocks at specific LBAs
    │
    ▼
Block Storage System (EBS / SAN / NVMe)  ←── stores raw 4KB blocks
    │
    ▼
Physical Disk (SSD / HDD)
```

The application accesses files through the file system. The file system translates file reads/writes to block addresses. Block storage handles fast, random access to individual blocks.

### Key Characteristics

**Random I/O**: can read or write any block at any position in microseconds. A database reading a single row doesn't read the whole file — it reads specific 8KB pages from their exact block addresses.

**Low latency**: NVMe SSD block storage achieves latencies of 100–500 microseconds for 4K random reads — orders of magnitude faster than object storage (10–50ms).

**Throughput and IOPS**:
```
EBS (AWS) volume types:
  gp3 (General Purpose SSD):  16,000 IOPS, 1,000 MB/s throughput
  io2 Block Express (Provisioned):  256,000 IOPS, 4,000 MB/s
  st1 (Throughput HDD):       500 MB/s throughput, low IOPS (large sequential reads)
  sc1 (Cold HDD):             250 MB/s throughput (archival, rarely accessed)
```

**Single attachment**: a block volume is typically attached to one VM at a time (read/write). Some systems (AWS EBS Multi-Attach) allow multiple VMs to attach a volume in read/write mode, but coordinating concurrent writes is the application's responsibility.

### AWS EBS Example

```python
import boto3

ec2 = boto3.client('ec2')

# Create a gp3 volume
volume = ec2.create_volume(
    AvailabilityZone='us-east-1a',
    Size=100,  # GB
    VolumeType='gp3',
    Iops=5000,        # provisioned IOPS
    Throughput=250,   # MB/s
    Encrypted=True
)

# Attach to EC2 instance
ec2.attach_volume(
    VolumeId=volume['VolumeId'],
    InstanceId='i-0abcdef123456',
    Device='/dev/sdf'
)

# On the instance: format and mount
# mkfs -t ext4 /dev/xvdf
# mount /dev/xvdf /data
```

### Snapshots

Block storage systems support **point-in-time snapshots** — capturing the state of an entire volume.

```
Volume (100 GB) at 2024-01-15 00:00 ──► Snapshot snap-001 (stores all 100 GB)
Volume changes during Jan 15         ──► Snapshot snap-002 (incremental — stores only changed blocks)
Volume changes during Jan 16         ──► Snapshot snap-003 (incremental)
```

EBS snapshots are stored in S3 (incremental after first snapshot). They can be used to:
- Restore a volume to a previous state.
- Create new volumes from the snapshot (in the same or different region).
- Clone a production database for testing.

### RAID for Performance and Redundancy

RAID (Redundant Array of Independent Disks) can be configured on top of multiple block volumes:

```
RAID 0 (Striping):
  Data striped across 4 volumes
  Throughput: 4× single volume
  Durability: if one volume fails, all data lost
  Use: temp data, scratch space, maximum performance

RAID 1 (Mirroring):
  Data mirrored on 2 volumes
  Throughput: read = 2×, write = 1×
  Durability: one volume failure is tolerated
  Use: databases (EBS already replicates internally)

RAID 10 (Mirror + Stripe):
  4 volumes: 2 mirrored pairs, striped
  Performance + redundancy
```

Note: EBS already replicates data within an AZ internally. RAID is rarely necessary unless you need IOPS that exceed a single EBS volume's limits.

### Block vs Object vs File Storage

| Dimension | Block | Object | File |
|---|---|---|---|
| Access method | Block addresses (file system on top) | HTTP REST API | Mount as network file system |
| Latency | Microseconds (SSD) | Milliseconds | Milliseconds (NFS) |
| Random I/O | Excellent | Poor | Good |
| Use case | Databases, VMs, OS disks | Static assets, backups, data lake | Shared access, home directories |
| Mount | Single VM (typically) | Not mounted — API access | Multiple VMs |
| Hierarchy | File system on top | Flat (prefix-simulated) | True directories |

---

## Key Trade-offs

| Advantage | Disadvantage |
|---|---|
| Lowest latency storage type | Not accessible without attaching to a VM |
| Random I/O performance for databases | More expensive than object storage |
| Full file system capabilities (POSIX) | Scales to single-volume capacity limits (EBS max: 64 TB) |
| Snapshots for backup and cloning | Not directly shareable across multiple VMs (usually) |

---

## When to use

- **Relational databases** (PostgreSQL, MySQL): require random IOPS for index lookups and page cache management.
- **Virtual machine OS disks**: the VM boot disk where the operating system and application binaries live.
- **NoSQL databases** (Cassandra, MongoDB): often run on local SSDs or EBS for storage.
- **Container persistent volumes** (Kubernetes PersistentVolumeClaims backed by EBS).
- **Any workload requiring low-latency random read/write access to data on disk**.

---

## Common Pitfalls

- **Not provisioning enough IOPS for databases**: databases are IOPS-intensive. Measure baseline IOPS (use CloudWatch DiskReadOps/DiskWriteOps) and provision 2× headroom. Depleting IOPS causes I/O wait and query latency spikes.
- **Not enabling delete-on-termination for ephemeral data volumes**: if a test VM is terminated but EBS volumes persist, you pay for unused storage indefinitely.
- **Ignoring throughput vs IOPS distinction**: gp3 IOPS and throughput can be configured independently. A workload doing large sequential reads (backup, analytics) needs high throughput MB/s, not high IOPS. Choose the right metric.
- **Running stateful workloads across AZs**: an EBS volume lives in one AZ. If you move a VM to another AZ, you can't attach the same volume. Plan your HA architecture with this constraint in mind.

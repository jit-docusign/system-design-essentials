# Apache ZooKeeper

## What it is

**Apache ZooKeeper** is a distributed coordination service that provides primitives for building higher-level distributed system constructs: configuration management, naming, synchronization, and group membership. It offers a simple, consistent hierarchical namespace (similar to a file system) that multiple distributed processes can reliably read and write to.

ZooKeeper was extracted from the Hadoop project and is now used as the coordination backbone for Kafka, HBase, HDFS (NameNode HA), and many other distributed systems. Kubernetes's equivalent is **etcd**.

---

## How it works

### ZooKeeper Data Model: znodes

ZooKeeper stores data in a tree of **znodes** — similar to a file system:

```
/
├── /config
│   ├── /config/database-url   → "postgres://db:5432/app"
│   └── /config/max-connections → "100"
├── /election
│   ├── /election/candidate-0000000001  (Node 1's ephemeral znode)
│   └── /election/candidate-0000000002  (Node 2's ephemeral znode)
└── /services
    └── /services/order-service
        ├── /services/order-service/instance-001 → "10.0.1.5:8080"
        └── /services/order-service/instance-002 → "10.0.1.6:8080"
```

Each znode can store a small data payload (typically < 1 MB, recommended < 1 KB).

### Znode Types

| Type | Description |
|---|---|
| **Persistent** | Remains until explicitly deleted. Config values, service registrations. |
| **Ephemeral** | Automatically deleted when the creating client session ends (disconnects/crashes). Used for leader election, presence detection. |
| **Sequential (Persistent or Ephemeral)** | ZooKeeper appends a unique monotonically increasing number to the znode name. Used for ordered locks and queuing. |

```python
from kazoo.client import KazooClient

zk = KazooClient(hosts='zookeeper:2181')
zk.start()

# Persistent znode
zk.create('/config/db-url', b'postgres://db:5432/app', makepath=True)

# Ephemeral znode (auto-deleted on client disconnect)
zk.create('/services/order-service/instance-001', b'10.0.1.5:8080',
          ephemeral=True)

# Sequential znode: creates /locks/order-lock-0000000042
path = zk.create('/locks/order-lock-', b'client-1', ephemeral=True,
                 sequence=True)
print(path)  # /locks/order-lock-0000000042
```

### Watches

A **watch** is a one-time notification triggered when a znode's data or children change:

```python
# Get data with a watch
data, stat = zk.get('/config/max-connections', watch=on_config_change)

def on_config_change(event):
    # Called when /config/max-connections changes
    new_value, _ = zk.get('/config/max-connections')
    reload_config(int(new_value))
    # Must re-register the watch for the next change
    zk.get('/config/max-connections', watch=on_config_change)
```

Watches are the foundation of ZooKeeper's coordination patterns — service discovery, distributed locks, and leader election all use watches to react to membership changes.

**Important**: watches fire exactly once and must be re-registered to continue monitoring.

### ZooKeeper Coordination Patterns

**Leader Election:**
```
Each node creates /election/leader- (ephemeral, sequential).
The node with the lowest sequence number is the leader.
Non-leaders watch the next-lower znode in the sequence.
When the watch fires → re-check sequence → promote if now lowest.
```

**Service Discovery:**
```
Each service instance creates an ephemeral znode:
  /services/order-service/10.0.1.5:8080 (auto-deleted on crash)

Service consumers:
  list children of /services/order-service → [10.0.1.5:8080, 10.0.1.6:8080]
  watch for children changes → refresh pool on registration/deregistration
```

**Distributed Configuration:**
```
Write to /config/feature-flag-x: true
All services with watches on this path are notified → hot-reload config
No service restarts needed.
```

**Distributed Lock:**
```
Create ephemeral sequential: /locks/process-0000000043
List /locks/ → if smallest → acquired
If not smallest → watch next-lower node
When next-lower deleted → re-check → acquire if now smallest
```

### ZAB Protocol (ZooKeeper Atomic Broadcast)

ZooKeeper uses its own consensus protocol called **ZAB** (ZooKeeper Atomic Broadcast) — predates Raft but serves the same purpose:

- **Leader** handles all writes; **followers** replicate.
- A quorum (majority) of nodes must acknowledge writes before they are committed.
- **Epoch numbers** prevent old leaders from interfering after re-election.
- Every committed transaction is assigned a globally unique **zxid** (ZooKeeper transaction ID) — monotonically increasing.

```
5-node ZooKeeper ensemble: quorum = 3
Write accepted if 3 of 5 nodes acknowledge
Tolerates 2 node failures (5-2 = 3 = quorum)
```

### ZooKeeper vs etcd

| Dimension | ZooKeeper | etcd |
|---|---|---|
| Consensus | ZAB | Raft |
| API style | Custom ZooKeeper API | gRPC / REST |
| Data model | Hierarchical znodes | Flat key-value (with range queries) |
| Watches | One-time, must re-register | Continuous watch (event stream) |
| Use cases | Kafka, HBase, Hadoop, legacy | Kubernetes, Consul, CoreOS ecosystem |
| Performance | ~10,000 writes/sec | ~10,000 writes/sec |
| Deployment | JVM-based (heavier) | Go binary (lighter) |
| Modern choice | Legacy/existing systems | New systems |

---

## Key Trade-offs

| Advantage | Disadvantage |
|---|---|
| Battle-tested coordination primitives | Complex watch semantics (one-time, must re-register) |
| Strong consistency (linearizable reads with sync) | Low throughput for write-heavy workloads |
| Ephemeral znodes for automatic presence detection | Single master bottleneck for writes |
| Widely supported in the Hadoop ecosystem | Not designed for large data storage |
| Mature and reliable | Heavier than etcd for Kubernetes-style use cases |

---

## When to use

ZooKeeper is appropriate for:
- **Kafka**: ZooKeeper (or KRaft in newer Kafka) manages broker metadata, controller election, and consumer group offsets.
- **HBase**: ZooKeeper coordinates region server assignment and Master HA.
- **Legacy distributed system coordination** where ZooKeeper is already the standard.

For new systems, **etcd** is generally preferred — simpler API, excellent Kubernetes integration, and continuous cluster watches without re-registration.

---

## Common Pitfalls

- **Storing large data in znodes**: ZooKeeper is not a key-value store for arbitrary data. Data payloads > 1MB degrade performance significantly. Store only small config values and pointers (IDs, addresses).
- **Not re-registering watches**: watches fire once and are removed. Forgetting to re-register in the watch callback means you miss subsequent changes.
- **Ensemble size**: ZooKeeper ensembles should have an odd number of nodes (3 or 5). A 4-node ensemble can only tolerate 1 failure (same as 3-node) — use 5 if you want to tolerate 2.
- **Large number of watches**: tens of thousands of outstanding watches on a single znode degrade NameNode/ZooKeeper performance under watch-storm events (all clients notified simultaneously). Limit watch registrations and use hierarchical fan-out.

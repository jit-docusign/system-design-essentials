# Leader Election

## What it is

**Leader election** is the process by which nodes in a distributed system agree on exactly one node to serve as the **leader** (or primary) — responsible for coordinating writes, running scheduled jobs, or managing shared resources. All other nodes are **followers** (replicas or standby).

Leader election is required when:
- A system needs a single authoritative coordinator to avoid conflicts.
- A primary node fails and a follower must take over without human intervention.
- Distributed tasks must be executed by exactly one node at a time.

---

## How it works

### Why We Need Leader Election

```
Without election (no agreed leader):
  All 3 nodes accept writes ──► each has a different view of "current state"
  Node 1: balance = $500
  Node 2: balance = $480 (received different writes)
  Node 3: balance = $510

With election (one leader):
  Node 1 is leader ──► all writes go to Node 1
  Node 2 and Node 3 replicate from Node 1
  Consistent view: balance = $500
```

### ZooKeeper-Based Leader Election

ZooKeeper's ephemeral sequential znodes provide a reliable election primitive:

```
1. Each candidate creates an ephemeral sequential znode:
   /election/candidate-0000000001
   /election/candidate-0000000002
   /election/candidate-0000000003

2. Each node lists all candidates and determines the smallest znode.

3. The node with the smallest ID is the leader.
   → Node holding candidate-0000000001 is the current leader.

4. All other nodes watch the next-smaller znode (not the leader directly):
   Node 2 watches Node 1
   Node 3 watches Node 2

5. If Node 1 crashes:
   → ephemeral znode /election/candidate-0000000001 is deleted automatically
   → Node 2 detects the watch fire → lists all znodes → is now smallest → becomes leader
   → Node 3 updates its watch to Node 2

Chain watching (vs. everyone watching the leader):
- Prevents the "herd effect" — all nodes racing to become leader simultaneously.
- Only the next-in-line gets notified and becomes the new leader.
```

```python
from kazoo.client import KazooClient
from kazoo.recipe.election import Election

# Kazoo (Python ZooKeeper client) election
zk = KazooClient(hosts='zookeeper:2181')
zk.start()

election = Election(zk, "/election", identifier=HOSTNAME)

def be_leader():
    """This function runs only while this node is leader."""
    while True:
        run_leader_tasks()
        time.sleep(1)

# Block here until this node becomes leader
election.run(be_leader)  # runs be_leader() when elected; blocks otherwise
```

### Raft Leader Election

The **Raft consensus algorithm** includes leader election as a core component:

```
All nodes start as Followers.

Election timeout fires (150–300ms random):
  Follower → Candidate
  Candidate increments term, votes for itself, sends RequestVote to all.

Majority votes received:
  Candidate → Leader

Leader sends periodic Heartbeats (AppendEntries with empty log):
  Heartbeat received → Follower resets election timeout

If no heartbeat within election timeout:
  Follower → Candidate → new election
```

Key properties:
- **Randomized timeouts** prevent split votes: 2 candidates don't always call elections simultaneously.
- **Term numbers**: each election is identified by a monotonically increasing term. Stale leaders with lower terms are rejected.
- **Majority vote**: only a candidate that can gather votes from a quorum (N/2 + 1) becomes leader — ensures at most one leader per term.

```
5-node cluster:  quorum = 3 votes needed to become leader
Node 1: term=5, votes=[1, 2, 3] → LEADER (3 ≥ 3)
Node 2: term=5, votes=[2, 4]    → loses (2 < 3)
```

### etcd-Based Leader Election (Kubernetes)

Kubernetes control plane components (scheduler, controller-manager) use **etcd leases** for leader election:

```python
# Kubernetes leader election (Go client):
import (
    leaderelection "k8s.io/client-go/tools/leaderelection"
    resourcelock "k8s.io/client-go/tools/leaderelection/resourcelock"
)

rl := &resourcelock.LeaseLock{
    LeaseMeta: metav1.ObjectMeta{Name: "my-controller", Namespace: "kube-system"},
    Client: clientset.CoordinationV1(),
    LockConfig: resourcelock.ResourceLockConfig{Identity: HOSTNAME},
}

leaderelection.RunOrDie(ctx, leaderelection.LeaderElectionConfig{
    Lock:          rl,
    LeaseDuration: 15 * time.Second,   // lock TTL
    RenewDeadline: 10 * time.Second,   // must renew within this
    RetryPeriod:   2 * time.Second,    // retry interval for non-leaders
    Callbacks: leaderelection.LeaderCallbacks{
        OnStartedLeading: func(ctx context.Context) { runLeaderWork(ctx) },
        OnStoppedLeading: func() { os.Exit(0) },
        OnNewLeader: func(identity string) { log.Infof("New leader: %s", identity) },
    },
})
```

### Handling Split Brain

**Split brain** occurs when two nodes both believe they are the leader — typically during a network partition:

```
Network partition:
  Partition A: [Node 1, Node 2]  — Node 1 is old leader, Node 2 elects Node 1 again
  Partition B: [Node 3, Node 4, Node 5] — majority partition elects Node 3 as leader

Both Node 1 and Node 3 act as leaders.
When partition heals → reconciliation needed.
```

Prevention with epoch/term fencing:
- Each leader has a term/epoch number.
- Any node that receives a message from a higher-term leader immediately demotes itself.
- Majority quorum requirement: Node 1 cannot maintain leadership without a majority — partition A has only 2 of 5 nodes, so Node 1 should step down.

---

## Key Trade-offs

| Advantage | Disadvantage |
|---|---|
| Eliminates conflicting writes from multiple primaries | Leader becomes a throughput bottleneck (single writer) |
| Simplifies consistency — all writes go to one node | Leader failure requires election time (seconds) = brief unavailability |
| Automatic failover without human intervention | Split brain can cause data divergence during partitions |
| Foundation for single-writer + multi-reader architectures | Adds coordination infrastructure (ZooKeeper, etcd) |

---

## When to use

Use leader election when:
- **Database primary/replica failover**: one node handles writes; followers replicate. On primary failure, elect a new primary automatically.
- **Scheduled job coordination**: a cronjob that must run on exactly one node (reports generation, nightly cleanups).
- **Shard management**: a single coordinator manages partition assignment to worker nodes (Kafka's Group Coordinator).
- **Distributed caches**: one leader manages cache invalidation or key assignment.

---

## Common Pitfalls

- **Not fencing the old leader**: after election, the old leader may still be running (not aware it lost leadership due to a network partition). Without fencing (epoch checks + rejecting writes from stale leaders), both old and new leaders accept writes — split brain.
- **Leases too short**: a leader with a 1-second lease must renew every 500ms. A GC pause of 600ms causes it to lose leadership. Size the lease to be at least 3–5× the P99 renewal latency.
- **Not handling `OnStoppedLeading`**: when a process loses leadership, it must stop doing leader work immediately. If it continues (unaware it lost the lease), split-brain effects occur.

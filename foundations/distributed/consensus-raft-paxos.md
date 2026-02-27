# Consensus: Raft and Paxos

## What it is

**Consensus** is the problem of getting multiple nodes in a distributed system to **agree on a single value or sequence of values** — even in the presence of node failures and network partitions. It is the foundational building block of distributed systems that require strong consistency: distributed databases, configuration stores, leader election, and replicated state machines.

**Paxos** (Leslie Lamport, 1989) and **Raft** (Diego Ongaro, 2014) are the two most widely known consensus algorithms. Raft was explicitly designed to be more understandable than Paxos; both achieve the same safety properties.

---

## How it works

### The Consensus Problem

```
5 nodes must agree on a value.

Node 1: "value is 42"
Node 2: "value is 17"
Node 3: crashes
Node 4: "value is 42"
Node 5: network partition

Challenge: reach agreement on exactly one value,
  despite crashes and message loss.
  
Properties required:
  Safety:     Only a value that was proposed is decided, and it's decided once.
  Liveness:   Eventually, a value is decided (system makes progress).
  Fault tolerance: Works with up to f failures out of 2f+1 nodes (majority quorum).
```

### Paxos

Paxos uses two phases and three roles: **Proposers**, **Acceptors**, **Learners**.

**Phase 1: Prepare**
```
Proposer sends "Prepare(n)" to all Acceptors (n = unique proposal number)

Acceptor responds with "Promise(n, v)" if:
  - n > any previously seen proposal number
  - Acceptor has already accepted a value v for a higher number → return it

Acceptor ignores if n ≤ highest seen number (prevents old proposals from interfering)
```

**Phase 2: Accept**
```
If Proposer receives Promise from majority (N/2 + 1):
  Proposer sends "Accept(n, v)" where:
    v = value from any returned Promise with highest number (if any)
    OR Proposer's own value if no prior value was returned

Acceptors accept the (n, v) if they haven't promised a higher number
```

**Quorum and safety**: because every majority quorum overlaps with every other majority quorum (in a 5-node cluster, any two quorums of 3 share at least 1 node), the algorithm guarantees that only one value can be chosen.

**Multi-Paxos** (for log replication): extends basic Paxos to agree on a sequence of log entries efficiently by reusing the Phase 1 preparation across multiple rounds.

### Raft

Raft is organized around a strong leader:

**Three roles**:
- **Leader**: handles all client requests; replicates log entries to followers.
- **Follower**: replicates from the leader; redirects client requests to the leader.
- **Candidate**: running for election when no heartbeat is received.

**Log Replication**:
```
Client ──► Leader: "SET x=5"

Leader:
  1. Appends entry to own log: [index=42, term=3, cmd="SET x=5"]
  2. Sends AppendEntries("SET x=5") to all followers
  3. Waits for majority (N/2+1) acknowledgment
  4. Commits: marks entry as committed in its log
  5. Replies SUCCESS to client
  6. Next AppendEntries heartbeat notifies followers of commit
  7. Followers execute "SET x=5" on their state machines
```

**Safety — The Log Matching Property**:
- If two logs have an entry with the same index and term, they are identical for all preceding entries.
- A leader never overwrites its own log — it only appends.
- A new leader is always chosen from candidates with the most up-to-date log (highest term, then highest index).

**Leader completeness**: once an entry is committed in term T, it will be present in all leaders of terms > T. This ensures committed values are never lost on leader change.

```
Term 1:  Leader = Node 1
  Log: [1:1: "SET x=5", 1:2: "SET y=3"]
  Both entries committed by majority.

Node 1 crashes. Election in Term 2:
  Node 3 becomes candidate with log: [1:1: "SET x=5", 1:2: "SET y=3"]  (up to date)
  Node 3 wins → new leader has all committed entries → correct

  Node 4 with log: [1:1: "SET x=5"] (missing one entry) cannot win:
  Voters with up-to-date logs will not vote for a behind candidate.
```

### Raft vs Paxos

| Dimension | Paxos | Raft |
|---|---|---|
| Understandability | Complex, many variants (Basic, Multi-Paxos, etc.) | Designed for clarity; easier to implement |
| Leader | Optional in basic Paxos; typically used in production | Explicit strong leader |
| Log replication | Less prescribed | First-class concept |
| Membership changes | Complex | Joint consensus / single-server changes |
| Used in | Chubby (Google), some Cassandra versions | etcd, CockroachDB, TiKV, Consul, RethinkDB |

### Real-World Usage

```
etcd (Kubernetes):
  Stores all K8s cluster state (pods, services, configmaps).
  3 or 5 node etcd cluster using Raft.
  Tolerate 1 failure (3-node) or 2 failures (5-node).

CockroachDB:
  Each data range is a 3–5 node Raft group.
  Thousands of Raft groups per cluster for multi-range distributed transactions.

Apache ZooKeeper:
  Uses ZAB (Zookeeper Atomic Broadcast) — similar to Raft, pre-dates Raft.
```

### Fault Tolerance

An N-node cluster can tolerate ⌊N/2⌋ failures:

| Cluster size | Quorum | Tolerated failures |
|---|---|---|
| 1 | 1 | 0 |
| 3 | 2 | 1 |
| 5 | 3 | 2 |
| 7 | 4 | 3 |

Adding nodes beyond 7 rarely improves availability but increases write latency (more nodes to acknowledge).

---

## Key Trade-offs

| Advantage | Disadvantage |
|---|---|
| Guarantees data consistency across replicas | Higher write latency (majority quorum round-trips) |
| Safe leader election and log replication | Liveness cannot be guaranteed under all conditions (FLP impossibility) |
| Well-understood fault tolerance guarantees | Requires odd number of nodes for clear majority |
| Foundation for etcd, Consul, CockroachDB | Performance scales poorly beyond 5–7 nodes per group |

---

## When to use

Consensus algorithms underlie:
- **Configuration stores**: etcd, Consul, ZooKeeper — storing cluster metadata that must be consistent.
- **Distributed databases**: CockroachDB, TiDB, Spanner — per-shard Raft groups for replication.
- **Leader election**: choosing a single primary node.
- **Service mesh control planes**: Istio, Consul use etcd/Raft for config distribution.

Application developers rarely implement Raft or Paxos directly — they use etcd, ZooKeeper, or a distributed database that runs consensus internally.

---

## Common Pitfalls

- **Two-node clusters**: a 2-node cluster has no majority when one fails (both must agree). Always use an odd number.
- **Slow network = consensus unavailability**: Raft requires majority acks within the election timeout. High latency cross-region clusters can trigger spurious elections. Tune election timeout = max(10× network RTT, 150ms).
- **etcd as a general-purpose store**: etcd is designed for small key-value metadata (< 1MB values, < a few GB total). Using it as an application database causes performance and capacity issues.

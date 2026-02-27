# Gossip Protocol

## What it is

A **gossip protocol** (also called epidemic protocol) is a **decentralized communication method** where nodes in a distributed system periodically exchange information with a small random subset of their peers — similar to how gossip (or a viral epidemic) spreads information through a network.

No single node has complete knowledge, no central coordinator is required, and yet information propagates to all nodes within O(log N) rounds — making gossip protocols highly fault-tolerant, scalable, and resistant to network partitions.

Used in: Cassandra (membership + ring topology), DynamoDB, Riak, Redis Cluster, Consul, Bitcoin (transaction propagation).

---

## How it works

### Core Loop

Every `T` milliseconds (gossip interval), each node:
1. Selects a small random set of peers (fanout F, typically 2–3).
2. Sends its current state (membership list, data version, chunk of state) to each selected peer.
3. Peers update their state based on received information and respond with their own state.

```python
import random
import time

class GossipNode:
    def __init__(self, node_id, peers):
        self.node_id = node_id
        self.peers = peers  # list of all known peers
        self.state = {node_id: {'alive': True, 'heartbeat': 0}}

    def gossip_round(self, fanout=2):
        # Pick fanout random peers
        targets = random.sample(self.peers, min(fanout, len(self.peers)))
        for target in targets:
            # Exchange state with target
            target.receive_gossip(self.state)

    def receive_gossip(self, incoming_state):
        for node_id, info in incoming_state.items():
            if node_id not in self.state:
                self.state[node_id] = info
            else:
                # Merge: use the state with the higher heartbeat
                if info['heartbeat'] > self.state[node_id]['heartbeat']:
                    self.state[node_id] = info
```

### Convergence Speed

With N nodes and fanout F, gossip converges in approximately `log_{F}(N)` rounds:

```
N=1000 nodes, F=2 fanout:
  Round 1: 1 node knows the info
  Round 2: ~3 nodes know (1 told 2 more)
  Round 3: ~7 nodes know
  ...
  Round 10: ~1023 nodes know (log₂(1000) ≈ 10)

100% of 1,000 nodes informed in ~10 rounds.
```

This is why gossip scales well — doubling nodes adds only one more round.

### Types of Gossip

**Anti-entropy gossip**: nodes exchange digests of their full state and sync differences. Ensures eventual consistency of all state.

```
Node A gossips digest (checksums) to Node B
Node B compares: "I'm missing key=X"
Node A sends full value of key=X to Node B
Node B updates its state
```

**Rumor mongering**: when a node learns new information ("hot news"), it enters rumor mode and actively pushes the update to random peers until it believes the information has spread.

```
Node A receives new membership event: "Node D joined"
Node A enters rumor mode: push event to B, C, E
B and C mark "Node D joined" as known → no longer in rumor mode
E still doesn't know → A or others will keep spreading
```

### Failure Detection with Gossip (SWIM Protocol)

Cassandra uses **SWIM (Scalable Weakly-consistent Infection-style Membership)** — a gossip-based failure detector:

```
Node A suspects Node B is dead (missed heartbeats):
  1. A sends "Ping" to B directly → no response (B is down or slow)
  2. A indirectly pings: sends Ping-Req to C, D, E → ask them to ping B
  3. C pings B → response → A informed B is alive (false positive avoided)
  4. No response via indirect ping → A marks B as SUSPECT
  5. Gossips SUSPECT status for B
  6. After suspect timeout → marks B as DEAD → gossips DEAD status

  On recovery: B gossips its own ALIVE status with a higher generation number
```

This indirect probing prevents false positives from transient network issues while detecting real failures within seconds.

### Cassandra's Gossip Usage

In Cassandra, every node runs the gossip protocol to maintain the **cluster membership ring**:

```
Each node maintains:
  - List of all nodes in the cluster
  - Each node's address, state (NORMAL/JOINING/LEAVING/DEAD), and load info
  - Each node's token ranges (virtual node assignment)

Gossip interval: every 1 second per node
Fanout: each node gossips to 1 live, 1 dead, 1 seed node per round
Convergence: ring state fully propagated in ~log(N) seconds
```

When a new node joins:
```
New Node C contacts seed node (statically configured):
  C ──► Seed: "I am C, token range [500-600]"
Seed gossips: "C has joined" to random peers
Within seconds: all nodes know C is in the ring → start routing requests to C
```

### Gossip-Based Data Dissemination (Scuttlebutt, CRDTs)

Gossip can propagate CRDT state — nodes exchange vector digests, and if a peer has a higher version of a key, it sends the delta. This is the **Scuttlebutt** protocol used in Dynamo, Riak, and CockroachDB for state synchronization.

---

## Key Trade-offs

| Advantage | Disadvantage |
|---|---|
| Highly fault-tolerant — no central coordinator | Eventual convergence (not immediate) |
| Scales to thousands of nodes (log N rounds) | Generates background network traffic (O(N × F) messages/round) |
| Works under network partitions (partially) | Information may temporarily diverge |
| Simple to implement and reason about | Not suitable for strong-consistency operations |
| Self-healing: nodes rejoin automatically | Malicious or faulty nodes can inject false state |

---

## When to use

Gossip protocols are appropriate for:
- **Cluster membership and failure detection**: Cassandra, Redis Cluster, Consul all use gossip for node presence.
- **Distributed key-value replication**: DynamoDB, Riak use gossip for anti-entropy and state sync.
- **Configuration broadcast**: propagating config changes to all nodes without a single coordinator.
- **Status aggregation in large clusters**: collecting health metrics, statistics across thousands of nodes.

Not suitable for:
- **Strong consistency operations**: gossip is eventually consistent — don't use it where immediate agreement is required.
- **Low-latency critical-path updates**: gossip has O(log N) propagation delay (seconds for large clusters) — not suitable for data the client is waiting for.

---

## Common Pitfalls

- **Too low gossip interval**: aggressive gossip (every 10ms) generates enormous network traffic at scale. Measure: N nodes × F fanout × state size × 100 rounds/sec × 2 directions.
- **Fanout too small**: F=1 is fragile if the one picked peer is down. Use F=2–3 for robust convergence.
- **Not seeding correctly**: gossip depends on seed nodes to introduce new nodes. If all seeds are down at startup, new nodes can't join. Configure multiple seeds and ensure they are highly available.
- **Trusting all gossip states**: in adversarial environments, a compromised node can gossip false membership or load information. Validate gossip state against signed certificates or use authenticated channels.

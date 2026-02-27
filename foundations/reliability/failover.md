# Failover Strategies

## What it is

**Failover** is the process by which a system automatically or manually switches to a backup component when the primary component fails. It is the operational mechanism that turns redundancy into actual availability — having two servers only helps if traffic can automatically shift from the broken one to the healthy one.

Failover applies at every layer of a system: application servers, databases, load balancers, DNS, and network hardware.

---

## How it works

The two primary failover architectures are:

### Active-Passive (Primary-Standby)

One node (the **primary**) handles all traffic. The other node (the **standby/passive**) is on standby — ready to take over but not serving traffic.

```
Normal operation:
  Clients → [Primary (Active)] ← health checks
                                   [Standby (Passive)] — replicating data, idle

Failure:
  [Primary] goes down
  Health check detects failure
  Clients → [Standby (now Active)] ← promoted
```

**Variants:**
- **Hot standby**: The passive node is fully running and up-to-date, can take over in seconds.
- **Warm standby**: The passive node is running but may be slightly behind; may take 1–5 minutes to promote.
- **Cold standby**: The passive is provisioned but not running; takes minutes to hours to bring online.

**Trade-offs:**
- Simpler to implement and reason about (no concurrent writes to reconcile).
- The standby is wasted capacity during normal operation (especially in cold standby).
- Failover time depends on how quickly the failure is detected.
- **Split-brain risk**: if the primary is slow (not dead) and failover triggers anyway, both nodes believe they are primary — this can corrupt state without fencing.

### Active-Active (Multi-Master)

All nodes handle traffic simultaneously. If one fails, the others absorb its load.

```
Normal operation:
  Clients → Load Balancer → [Node A] [Node B] [Node C] ← all serving traffic

Failure:
  [Node B] goes down
  Load Balancer health check removes Node B
  Clients → [Node A] [Node C] ← share increased load
```

**Trade-offs:**
- Better resource utilization — all servers contribute to throughput.
- No single standby needed — capacity is inherent in the cluster size.
- Harder to maintain consistency — concurrent writes to multiple active nodes can conflict.
- Requires conflict resolution or write routing to avoid inconsistencies.

### Failure Detection

Failover begins with detecting that a node has failed. Common mechanisms:

| Method | Description |
|---|---|
| **Health checks** | Load balancer or orchestrator pings nodes on a defined endpoint; removes them if they fail |
| **Heartbeat** | Nodes send periodic signals to a coordinator; absence of signal triggers failover |
| **Gossip protocol** | Nodes communicate peer failure state to each other without centralized coordination |
| **Quorum-based** | Nodes vote; if a quorum doesn't hear from a node, it's declared dead |

### DNS-based Failover

For cross-region or cross-datacenter failover, DNS TTLs are lowered and DNS records are updated to point to the backup region. The key constraint is TTL (Time To Live) — if the TTL is 300s, clients may still route to the failed primary for up to 5 minutes after failover.

### Database Failover

Databases require special handling because failover must not lead to split-brain or data loss:

1. **Primary fails** → replica is promoted to primary.
2. **Promote sequence**: stop all writes to old primary → wait for replication to catch up → elect new primary → update routing → restart old primary as replica.
3. **Fencing tokens**: before promoting a replica, issue a token that the old primary must use on future writes; if it presents an old token it is rejected, preventing split-brain.

---

## Key Trade-offs

| | Active-Passive | Active-Active |
|---|---|---|
| **Simplicity** | High | Low (conflict resolution) |
| **Resource efficiency** | Low (standby is idle) | High (all nodes serve traffic) |
| **Consistency** | Easier (single writer) | Harder (multi-writer conflicts) |
| **Failover speed** | Seconds to minutes | Sub-second (traffic absorbed) |
| **Capacity during failure** | Reduced (single node handles all) | Gracefully reduced |

---

## When to use

**Active-Passive:**
- Databases where write consistency is critical (relational DBs, config stores).
- Systems where conflict resolution on multi-master writes is too complex.
- Simpler operational model; lower concurrency requirements.

**Active-Active:**
- High-traffic stateless application layers (web servers, API servers).
- Globally distributed systems needing low-latency local writes.
- Databases that support multi-master with last-write-wins or CRDT-based conflict resolution.

---

## Common Pitfalls

- **Split-brain**: The most dangerous failure mode — two nodes each believe they are the primary and accept writes, causing divergent state. Always use fencing tokens or lease-based mechanisms.
- **Failover time longer than expected**: Health check intervals + detection timeout + promotion time adds up. A check every 30s with 3 failures required = 90s before failover begins. Tune these values deliberately.
- **Failing over unnecessarily (false positives)**: A slow but healthy node triggering failover unnecessarily causes disruption. Add hysteresis (require multiple consecutive failures before declaring a node dead).
- **Forgetting DNS TTL**: Low-level infrastructure failover may complete in seconds, but if DNS TTL is 300s, clients will keep hitting the failed IP.
- **Not testing failover regularly**: Failover paths that are never exercised rot. Run regular game days or chaos engineering exercises to ensure failover actually works.

# Redundancy

## What it is

**Redundancy** is the practice of adding extra capacity, components, or copies of data beyond what is strictly needed for normal operation. The surplus exists solely to absorb failures — when the primary component fails, the redundant components take over.

Redundancy is the foundational strategy for achieving high availability. Without redundancy, any single failure produces an outage. With correctly designed redundancy, the system tolerates component failures without users noticing.

---

## How it works

Redundancy applies at every layer of a system. A complete high-availability design eliminates single points of failure (SPOFs) at all levels.

### Hardware Level

- **Redundant power supplies**: servers have dual PSUs connected to different power circuits.
- **RAID storage**: data is striped/mirrored across multiple disks so a single disk failure doesn't lose data.
- **Redundant NICs**: multiple network cards for failover if one fails.

### Network Level

- **Multiple ISP uplinks**: connect to two or more internet service providers so a single provider outage doesn't disconnect the datacenter.
- **Redundant switches and routers**: no single network device is in the only path.
- **BGP anycast**: route traffic to the nearest available datacenter — if one goes down, routes converge to another.

### Application Server Level

- **Multiple instances behind a load balancer**: if one server crashes, the load balancer routes traffic to the remaining healthy servers.
- **Auto-scaling groups**: automatically replace failed instances with new ones.
- **Blue-green / canary deployments**: keep the old version running while rolling out the new one, so a bad deploy can be instantly rolled back.

### Database Level

- **Primary-replica replication**: the replica is a redundant copy that can be promoted if the primary fails.
- **Multi-region replication**: a copy of the data in a separate geographic region for disaster recovery.
- **Backups**: a time-delayed redundant copy that protects against logical errors (data corruption, accidental deletion) that would replicate immediately to a live replica.

### Geographic Level

- **Multiple availability zones (AZs)**: distribute resources across physically separate datacenters within a region; AZ-level failures (power, cooling) don't take down the whole system.
- **Multiple regions**: protect against regional disasters (natural disasters, complete datacenter loss). Comes at the cost of cross-region replication latency.

### Redundancy Models

**N+1 Redundancy**: Run $N$ instances needed to handle load, plus 1 spare. If one fails, the spare absorbs the load.

**N+2 Redundancy**: Tolerates two simultaneous failures — used in highly critical systems.

**2N Redundancy (Active-Active / Full Redundancy)**: Run two complete duplicate systems simultaneously. Each can absorb the full load. Most expensive; used where even brief degradation is unacceptable.

### Standby Types

| Standby Type | State | Failover Time | Cost |
|---|---|---|---|
| **Hot standby** | Fully running, data in sync | Seconds | High (resource cost of running duplicate) |
| **Warm standby** | Running but may have replication lag | Minutes | Medium |
| **Cold standby** | Provisioned/stopped; not running | Minutes to hours | Low |

### Availability Math: Series vs Parallel

With redundancy, components in parallel dramatically improve availability:

**Series** (all must work):
$$A_{\text{total}} = A_1 \times A_2 \times A_3$$

Two 99.9% components in series: $0.999 \times 0.999 = 99.8\%$ — *worse* than either component alone.

**Parallel** (at least one must work):
$$A_{\text{total}} = 1 - (1 - A_1)(1 - A_2)$$

Two 99.9% components in parallel: $1 - (0.001 \times 0.001) = 99.9999\%$ — dramatically better.

This is why redundancy is placed at each layer independently — you want parallel paths at every potential failure point.

---

## Key Trade-offs

| Trade-off | Description |
|---|---|
| **Cost** | Redundant components cost money. Each additional "nine" of availability roughly doubles infrastructure cost. |
| **Complexity** | More components, more failure modes to design for. Load balancers, health checks, failover logic all add complexity. |
| **Consistency** | Multiple replicas of data can diverge, especially with async replication. Redundancy and consistency require careful coordination. |
| **Diminishing returns** | Moving from 99.9% to 99.99% is valuable; 99.999% to 99.9999% is rarely worth the enormous incremental cost. |

---

## When to apply

- **Every user-facing service** should have at minimum N+1 redundancy at the application layer. A single server handling all traffic is always a SPOF.
- **Database primary** should always have replicas for both failover and read scaling.
- **Critical dependencies** (auth services, payment gateways, DNS) should have cross-AZ redundancy at minimum.
- **Recovery Time Objective (RTO) and Recovery Point Objective (RPO)** drive the choice of standby type — if you can tolerate 30 minutes of downtime, cold standby may be sufficient; if you can tolerate none, hot standby is required.

---

## Common Pitfalls

- **Redundancy that shares a failure domain**: Two servers in the same rack share the same power and network — a rack failure takes down both. True redundancy requires different failure domains (separate racks, separate AZs, separate regions).
- **Backup is not redundancy**: Backups protect against logical errors and data loss but take time to restore. A replica handles failover in seconds; a backup restoration may take hours. Both are needed for different failure modes.
- **SPOF hiding in surprising places**: Applications often have hidden SPOFs — a single DNS entry, a single centralized config server, a single certificate authority. Map your architecture and systematically find and eliminate all SPOFs.
- **Not testing failover**: Redundant systems that are never exercised often fail when needed. Regularly test failover paths to ensure they work and teams know the procedures.
- **Over-provisioning redundancy for non-critical systems**: Spending engineering effort achieving five-nines availability on an internal dev tool is waste. Match redundancy level to the business criticality and cost of downtime.

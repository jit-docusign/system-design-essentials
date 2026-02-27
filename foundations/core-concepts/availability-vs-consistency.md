# Availability vs Consistency

## What it is

In distributed systems, **availability** means every request receives a response — even if that response might not reflect the most recent write. **Consistency** means every read returns the most recent write, or returns an error.

These two properties are in direct tension when a network partition occurs, but the trade-off exists even in the happy path: achieving strong consistency requires coordination between nodes, which costs latency and availability.

This is the central design question of almost every distributed system: *when something goes wrong — or even when things are running normally — should the system prefer to stay available, or to stay correct?*

---

## How it works

### Availability

**Availability** is typically expressed as a percentage of time a system is operational and responds successfully:

| Availability | Downtime per year | Downtime per month |
|---|---|---|
| 99% ("two nines") | 3.65 days | 7.3 hours |
| 99.9% ("three nines") | 8.76 hours | 43.8 minutes |
| 99.99% ("four nines") | 52.6 minutes | 4.4 minutes |
| 99.999% ("five nines") | 5.26 minutes | 26 seconds |

Each additional "nine" requires roughly 10x more engineering effort. Chasing five nines unconditionally is often not worth it — align availability targets with the actual business cost of downtime.

For systems composed of multiple components, availability compounds:

$$A_{\text{series}} = A_1 \times A_2 \times \cdots \times A_n$$

A system with 5 components each at 99.9% availability has an overall availability of $0.999^5 \approx 99.5\%$.

Redundancy reverses this:

$$A_{\text{parallel}} = 1 - (1 - A_1)(1 - A_2)$$

Two components at 99.9% running in parallel: $1 - (0.001)^2 = 99.9999\%$.

### Consistency

In distributed systems, consistency has multiple levels (the **consistency spectrum**):

| Level | Description | Example |
|---|---|---|
| **Strong consistency** | Every read reflects the latest write | Traditional RDBMS, Zookeeper |
| **Sequential consistency** | All nodes see operations in the same order, not necessarily real-time | Some distributed KV stores |
| **Causal consistency** | Operations that are causally related are seen in the correct order | Social media replies after posts |
| **Read-your-writes** | A client always sees its own writes | Session-level consistency |
| **Monotonic reads** | Once a client reads a value, it never reads an older one | Prevents seeing data go "backward" |
| **Eventual consistency** | Given no new writes, all replicas converge to the same value | DNS, Cassandra (with default settings) |

### The Core Tension

Strong consistency requires that before a write is acknowledged, it must be confirmed by a quorum of nodes. This means:
- If nodes can't communicate (partition), you must either **wait** (sacrificing availability) or **respond with potentially stale data** (sacrificing consistency).
- Even without partitions, cross-node coordination adds latency, reducing throughput.

---

## Key Trade-offs

| Preference | What you gain | What you give up |
|---|---|---|
| **Consistency** | Always correct data; no conflicting versions | Higher latency; system may refuse requests during partition |
| **Availability** | Always responds; works through failures | Data may be stale or conflicting; reconciliation needed later |

**Real-world examples:**
- **Banking / inventory** → favor consistency; a wrong balance or oversold ticket is worse than a failed transaction.
- **Social media likes / view counts** → favor availability; showing a slightly stale count is acceptable.
- **Collaborative editing** → complex; requires CRDT or OT to be both available and eventually consistent.
- **DNS** → strongly availability-favoring; propagation lag (stale data) is acceptable.

---

## When to use each

**Choose strong consistency when:**
- Correctness of data is non-negotiable (financial transactions, inventory, bookings)
- Conflicts are expensive or impossible to resolve after the fact
- Users/systems depend on reading their own writes immediately

**Choose availability (eventual consistency) when:**
- Temporary staleness is acceptable
- The system must continue operating during network instability
- The data can be reconciled or merged later (shopping carts, counters, collaborative docs)

---

## Common Pitfalls

- **Assuming "eventually consistent" means "probably fine"**: Eventual consistency without proper conflict resolution can lead to permanent data loss or incorrect final states.
- **Ignoring read-your-writes**: A user submits a form, then immediately reads and doesn't see their change — this feels broken even if the system is "eventually consistent."
- **Conflating availability with reliability**: An available system returns responses even if wrong; a reliable system only returns correct responses. These are distinct goals.
- **Not modeling failure modes**: Consistency/availability trade-offs only matter when something fails. Most systems work fine under both models — design for what happens in the 1% failure case.
- **Treating consistency as binary**: There are many levels on the consistency spectrum. Requiring strictly serializable behavior everywhere is unnecessarily expensive; pick the weakest consistency level that is still correct for each use case.

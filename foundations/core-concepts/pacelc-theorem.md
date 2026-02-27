# PACELC Theorem

## What it is

The **PACELC Theorem**, proposed by Daniel Abadi in 2010, extends the CAP Theorem to address a critical gap: **CAP only describes behavior during a network partition, but most of the time systems are running normally without a partition.** What happens then?

PACELC states:

> In the case of a **P**artition (**P**), a system must choose between **A**vailability and **C**onsistency (the CAP trade-off); **E**lse (**E**), even when the system is running normally without a partition, it must choose between **L**atency and **C**onsistency.

The theorem can be written as:

**PAC | ELC**

This is why PACELC is a more complete model for classifying real-world distributed databases.

---

## How it works

### The Two Scenarios

**Scenario 1: Partition (P) → Choose A or C**

Same as CAP. When nodes can't communicate, do you return (possibly stale) data, or do you refuse to respond until consistency can be guaranteed?

**Scenario 2: No Partition (Else/E) → Choose Latency (L) or Consistency (C)**

Even when the network is healthy, achieving strong consistency requires coordination:
- A write must be acknowledged by a quorum of replicas before confirming to the client.
- A read must verify it has the latest value from a quorum.

This coordination adds **latency**. If you want low latency, you must accept the possibility of reading slightly stale data (weaker consistency).

### PACELC Classification of Common Systems

| System | Partition behavior | Normal operation | Classification |
|---|---|---|---|
| DynamoDB (default) | Available | Low latency | PA / EL |
| Cassandra (default) | Available | Low latency | PA / EL |
| Zookeeper | Consistent | Lower latency due to CP design | PC / EC |
| HBase | Consistent | Consistent (waits for ACK) | PC / EC |
| MongoDB (default) | Consistent | Consistent | PC / EC |
| DynamoDB (strong reads) | Consistent | Consistent | PC / EC |
| CRDT-based systems (e.g. Riak) | Available | Low latency | PA / EL |

### The Normal-Operation Trade-off in Depth

Consider a system with 3 replicas and a write quorum of 2:

```
Write: "balance = $500"

        ┌──────────────────────────────────────────────────┐
Client  │  → Node 1 (primary)                              │
        │      → Node 2 (replica) ← wait for ACK          │ High consistency,
        │      → Node 3 (replica) ← (optional quorum)     │ Higher latency
        └──────────────────────────────────────────────────┘
                             vs
        ┌──────────────────────────────────────────────────┐
Client  │  → Node 1 (primary) → ACK immediately           │
        │      → Node 2 (async replication)               │ Low latency,
        │      → Node 3 (async replication)               │ Weaker consistency
        └──────────────────────────────────────────────────┘
```

The first approach (synchronous quorum) gives strong consistency but adds the round-trip time to replicas before acknowledging the client. The second approach (async replication) responds immediately but means a subsequent read from Node 2 might see stale data.

---

## Key Trade-offs

| Choice | What you get | What you give up |
|---|---|---|
| **EL** (low latency) | Fast reads/writes in the normal case | Possible stale reads; replication lag |
| **EC** (high consistency) | Every read reflects the latest write | Added latency from synchronous coordination |

---

## Why PACELC is More Useful Than CAP Alone

1. **Partitions are rare.** The latency/consistency trade-off in the *normal case* affects every request — it's far more important day-to-day than the partition scenario.
2. **Database designers explicitly tune for EL or EC.** Cassandra's tunable consistency (ONE, QUORUM, ALL) is essentially a dial between EL and EC.
3. **It explains why "AP" databases like Cassandra still have consistency knobs.** Even an AP/PA system can provide EC behavior for reads/writes if you configure it to use QUORUM.

---

## When to apply

Use PACELC when:
- Choosing or evaluating a database for a latency-sensitive workload.
- Deciding on consistency levels for a Cassandra or DynamoDB table.
- Explaining why reducing replication lag requires accepting eventual consistency.
- Designing a system where both partition resilience *and* normal-operation latency matter.

---

## Common Pitfalls

- **Stopping at CAP**: CAP tells you what happens during a partition. PACELC tells you what your system behaves like 99.9% of the time — which is equally important.
- **Assuming synchronous replication is always "better"**: It provides stronger consistency but at a significant latency and availability cost. Many production systems are intentionally EL.
- **Ignoring tunable consistency**: Many systems (Cassandra, DynamoDB) are PA/EL by default but can approximate PC/EC behavior for specific operations — understanding PACELC helps you make that decision consciously.
- **Not auditing your stack's PACELC positions**: Most systems use several storage layers. An API that reads from an EL cache, falls back to an EL replica, and writes to an EC primary has a complex composite PACELC position that may not be obvious from any single component's documentation.
- **Not communicating the trade-off to stakeholders**: Choosing EL means some users will occasionally read stale data. Teams that don't understand this ship bugs or file incorrect "data loss" incidents.

# CAP Theorem

## What it is

The **CAP Theorem**, formulated by Eric Brewer in 2000 and formally proven by Gilbert and Lynch in 2002, states that a distributed data store can only guarantee **two of the three** following properties simultaneously:

- **C — Consistency**: Every read receives the most recent write, or an error.
- **A — Availability**: Every request receives a non-error response (though it may not be the most recent data).
- **P — Partition Tolerance**: The system continues to operate despite network messages being dropped or delayed between nodes.

In practice, **network partitions are not optional** — any networked system must tolerate them. This means the real choice is always between **Consistency and Availability** when a partition occurs.

> **Important precision**: The "C" in CAP is specifically **linearizability** — the guarantee that once a write is acknowledged, any subsequent read from any node returns that value. It is not serializability (transaction-level ordering), and it is not the same "C" as in ACID (constraint satisfaction). This distinction matters when evaluating vendor claims of "CAP consistency".

---

## How it works

### Why P is non-negotiable

A network partition is when nodes in a distributed system cannot communicate with each other. This happens regularly in production — due to hardware failures, network congestion, misconfiguration, or data center incidents.

If you build a system that cannot tolerate partitions (sacrifices P), it means the system assumes the network is always reliable. This is an unrealistic and dangerous assumption for any system running across multiple machines.

Therefore: **every real distributed system is either CP or AP**.

### CP Systems (Consistency + Partition Tolerance)

When a partition occurs, CP systems choose to **return an error or time out** rather than return potentially stale data.

```
Node A ←──✗ (partition) ✗──→ Node B

Client: "What is the balance?"
Node A: "I can't confirm with Node B. Returning error."
```

- **Characteristic**: Strong consistency guarantees; refuses requests it can't serve correctly.
- **Examples**: HBase, Zookeeper, etcd, traditional RDBMS clusters (with quorum writes)
- **Use when**: Data correctness is critical — financial systems, inventory, coordination services.

### AP Systems (Availability + Partition Tolerance)

When a partition occurs, AP systems choose to **continue serving requests** — potentially returning stale or conflicting data.

```
Node A ←──✗ (partition) ✗──→ Node B

Client: "What is the balance?"
Node A: "Last I knew, it was $500." (may be stale)
```

- **Characteristic**: Always responds; data may be inconsistent across nodes until partition heals.
- **Examples**: Cassandra (with default consistency), DynamoDB, CouchDB, DNS
- **Use when**: Service availability is critical and temporary staleness is acceptable.

### Visualizing the spectrum

```
                  Consistency
                       │
                    CP │
                       │
Partition─────────────────────────── (CA: only in single-node or non-distributed)
Tolerance              │
                    AP │
                       │
                  Availability
```

Note: **CA systems** (sacrificing partition tolerance) exist only in single-node or non-distributed deployments — effectively, a traditional single-server RDBMS. Once you go distributed, P is mandatory.

---

## Key Trade-offs

| System Type | Partition behavior | Use case |
|---|---|---|
| **CP** | Rejects or blocks requests until partition heals | Financial transactions, distributed locks, config management |
| **AP** | Continues serving, may return stale data | Shopping carts, social feeds, DNS, analytics events |

---

## Limitations of CAP

CAP is a useful mental model but has limitations:

1. **It's binary, but reality is a spectrum**: Consistency isn't all-or-nothing. There are many levels (see [Availability vs Consistency](availability-vs-consistency.md)).
2. **It only addresses partitions**: It says nothing about the latency/consistency trade-off under *normal* operation. PACELC addresses this gap.
3. **"Available" is loosely defined**: CAP's definition of availability (any response) doesn't distinguish between a correct stale response and a completely wrong one.
4. **Partitions are rare, but their handling still matters**: Most CAP discussion focuses on the partition case, but the design choices made for partitions affect everyday behavior.
5. **The CP/AP binary is often misleading in practice**: Martin Kleppmann's critique ("A Critique of the CAP Theorem", 2015) demonstrates that the CP/AP categorisation breaks down for many real systems. A system labelled "CP" might still return stale data under some operations. A system labelled "AP" might provide linearizable behaviour on specific read paths depending on configuration. The label is a useful starting point, not a precise specification of behaviour. When correctness actually matters, you need to reason about the specific guarantees of the specific operations you are using — not the CAP label on the tin.

---

## When to use it

CAP Theorem is most useful as a **conversation starter and classification tool** rather than a precise design specification. Use it to:
- Identify whether a system should prioritize correctness or uptime during network failures.
- Categorize existing systems ("Cassandra is AP; Zookeeper is CP").
- Communicate trade-off decisions to stakeholders.

For more nuanced design, use PACELC, consistency levels, and quorum configurations.

---

## Common Pitfalls

- **Treating CA as a real option for distributed systems**: It isn't. If you're distributing data across machines, you must tolerate partitions.
- **Thinking CP means "always consistent"**: CP databases are only consistent when they agree to respond — they may still have eventual consistency in replicas or allow tunable consistency levels.
- **Ignoring the latency dimension**: A CP system that takes 30 seconds to respond "correctly" during a partition may be worse than an AP system that returns slightly stale data instantly.
- **Applying CAP at the system level, not the operation level**: Many databases expose both CP and AP behavior depending on the operation or consistency level chosen per query (e.g., Cassandra).
- **Taking CP/AP labels at face value**: As the Kleppmann critique shows, the label is an approximation. Verify the actual consistency guarantees of the operations your system depends on, not the marketing classification.
- **Over-relying on CAP as a final answer**: It's a theorem about a narrow trade-off. Real system design requires understanding much more — replication strategies, failure modes, SLAs, and consistency models.

# Back-of-the-Envelope Estimation

## What it is

**Back-of-the-envelope estimation** is the practice of rapidly deriving approximate figures for system capacity requirements — storage, throughput, bandwidth, memory — using simple arithmetic and known reference numbers. The goal is not precision, but **order-of-magnitude correctness** sufficient to validate design decisions, identify bottlenecks early, and communicate feasibility.

This skill separates engineers who design systems that will actually work at scale from those who guess. In system design discussions, it is used to answer questions like: "How many servers do we need?", "Will this fit in memory?", "Can a single database handle this write load?"

---

## How it works

### Step 1: Know your reference numbers

Memorize a handful of numbers that serve as anchors for all estimation:

**Hardware / Compute:**
| Resource | Approximate |
|---|---|
| L1 cache access | 1 ns |
| Main memory (RAM) access | 100 ns |
| SSD random read | 100 µs |
| HDD sequential read | 100 MB/s |
| SSD sequential read | 500 MB/s – 3 GB/s |
| Network within same datacenter | ~0.5 ms RTT |
| Network cross-continent | ~100–150 ms RTT |

**Storage:**
| Unit | Value |
|---|---|
| 1 KB | 10³ bytes |
| 1 MB | 10⁶ bytes |
| 1 GB | 10⁹ bytes |
| 1 TB | 10¹² bytes |
| 1 PB | 10¹⁵ bytes |

**Time:**
| Period | Seconds |
|---|---|
| 1 minute | 60 s |
| 1 hour | 3,600 s |
| 1 day | 86,400 s ≈ 10⁵ s |
| 1 month | ~2.6 × 10⁶ s |
| 1 year | ~3.15 × 10⁷ s ≈ 3 × 10⁷ s |

**Throughput rule of thumb:** 1 server can handle ~1,000–10,000 RPS for lightweight operations (varies enormously by operation cost).

---

### Step 2: Establish scale assumptions

State your assumptions explicitly before calculating:

- **Daily Active Users (DAU)**: e.g. 100 million
- **Read/write ratio**: e.g. 10:1
- **Peak multiplier**: peak traffic is typically 2–3× average
- **Retention window**: how long data is stored

---

### Step 3: Derive QPS

$$\text{QPS} = \frac{\text{DAU} \times \text{requests per user per day}}{86{,}400 \text{ s}}$$

**Example**: 100M DAU, each making 10 reads and 1 write per day:
- Read QPS: $\frac{100M \times 10}{86{,}400} \approx 11{,}574 \text{ RPS} \approx$ **~12K read RPS**
- Write QPS: $\frac{100M \times 1}{86{,}400} \approx 1{,}157 \text{ RPS} \approx$ **~1.2K write RPS**
- Peak (3×): ~36K read RPS, ~3.5K write RPS

---

### Step 4: Estimate storage

$$\text{Storage per day} = \text{Write QPS} \times 86{,}400 \times \text{bytes per write}$$

**Example**: 1.2K write RPS, avg write size = 500 bytes:
- $1{,}200 \times 86{,}400 \times 500 \approx 51.8 \text{ GB/day}$
- Over 5 years: $51.8 \times 365 \times 5 \approx$ **~94.5 TB**

---

### Step 5: Estimate bandwidth

$$\text{Bandwidth} = \text{QPS} \times \text{avg payload size}$$

**Example**: 12K read RPS, avg response = 10 KB:
- $12{,}000 \times 10{,}000 = 120 \times 10^6 \text{ bytes/s} =$ **~120 MB/s incoming bandwidth**

---

### Step 6: Estimate server count

$$\text{Servers} = \frac{\text{Peak QPS}}{\text{QPS per server}}$$

**Example**: Peak 36K read RPS, each server handles 3K RPS:
- $36{,}000 / 3{,}000 = 12$ application servers (add headroom → round up to 15–20)

---

### Step 7: Translate to cost

Estimating resources without connecting them to cost misses half the value. Use rough reference numbers to sense-check architecture decisions:

| Resource | Rough unit cost (AWS rough order of magnitude) |
|---|---|
| General-purpose compute (m5.xlarge, 4 vCPU / 16 GB) | ~$150/month |
| Storage: SSD EBS | ~$0.10 / GB-month |
| Storage: S3 object storage | ~$0.023 / GB-month |
| Storage: Glacier archival | ~$0.004 / GB-month |
| Data transfer out (internet) | ~$0.09 / GB |
| Managed cache (ElastiCache r6g.large, 13 GB RAM) | ~$150/month |
| Managed DB (RDS db.m5.xlarge, Multi-AZ) | ~$500/month |

*Prices vary by region, reserved vs on-demand, and negotiated rates. These are for order-of-magnitude reasoning only.*

**Example**: 95 TB of user data over 5 years.
- On SSD: 95,000 GB × $0.10 = **$9,500/month** in storage alone — a signal to evaluate tiered storage.
- Move data older than 90 days to S3 ($0.023/GB): if 80% of data is cold, active SSD tier = 19 TB ($1,900/month), cold S3 tier = 76 TB ($1,748/month) — total ~$3,650/month. Tiering saves ~60%.
- At 76 TB cold, if access pattern further allows archival after 1 year: Glacier at $0.004/GB changes the math again.

Cost estimation like this is how architectural decisions get made in practice: a back-of-the-envelope that reveals tiering is necessary is worth more than a polished diagram that ignores the storage bill.

**Assumptions**: 100M DAU; 1 write per day (create short URL), 10 reads per day (redirect).

| Metric | Calculation | Result |
|---|---|---|
| Write QPS | 100M / 86,400 | ~1,200 RPS |
| Read QPS | 100M × 10 / 86,400 | ~11,600 RPS |
| Bytes per short URL entry | 7 (short code) + 256 (long URL) + metadata ≈ 500 B | 500 B |
| Daily storage | 1,200 × 86,400 × 500 B | ~52 GB/day |
| 5-year storage | 52 GB × 365 × 5 | ~95 TB |
| Read bandwidth | 11,600 × 500 B | ~5.8 MB/s |
| Servers (at 5K RPS each) | 11,600 / 5,000 | ~3 (with headroom: 5) |

---

## Key Trade-offs

Estimation is about **communicating reasoning**, not about exact numbers. A 2× error is generally acceptable. A 10× error in either direction suggests a wrong assumption worth revisiting.

Always state your assumptions first — the assumptions matter more than the arithmetic.

---

## Common Pitfalls

- **Skipping estimation and guessing**: Informal "it should be fine" assessments miss order-of-magnitude problems before they reach production.
- **Not stating assumptions**: Calculations without stated assumptions cannot be challenged or corrected.
- **Forgetting peak multipliers**: Designing for average load is a recipe for outages. Always apply a 2–3× peak multiplier.
- **Missing the replication multiplier for storage**: Data stored in 3 replicas costs 3× the raw data size. Always account for replication factor.
- **Ignoring metadata and indexing overhead**: The raw data size is not the only storage cost. Indexes, WAL, and metadata can 1.5–3× the actual stored size in some systems.
- **Getting lost in arithmetic precision**: Round aggressively ($86{,}400 \approx 10^5$), state the rounding, and move on. The value is in the order-of-magnitude reasoning, not the exact digits.
- **Stopping at resource counts without computing cost**: An estimate that yields "we need 200 TB of SSD" without translating to a monthly bill misses the decision it should enable — whether tiering, compression, or a different storage model is worth it.
- **Not using estimates to foreclose options**: The value of estimation is knowing early when an assumption breaks. An estimate that shows you need 10 TB/day of writes immediately tells you a single-node database won't work — sharding or a different storage model must be baked in from the start.

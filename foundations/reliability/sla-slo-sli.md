# SLA / SLO / SLI

## What it is

**SLA, SLO, and SLI** form a three-layer framework for defining, measuring, and agreeing on service reliability. They turn the vague goal of "high availability" into concrete, measurable, and contractually enforceable commitments.

| Term | Full Name | What it is |
|---|---|---|
| **SLI** | Service Level Indicator | A specific metric that measures a dimension of service quality |
| **SLO** | Service Level Objective | An internal target for an SLI (e.g., "P99 latency ≤ 200ms") |
| **SLA** | Service Level Agreement | A contractual commitment to external customers, with consequences for breaches |

The relationship: **SLI** is what you measure → **SLO** is what you target → **SLA** is what you promise.

---

## How it works

### SLI — Service Level Indicator

An SLI is a quantitative measurement of a service behavior. Examples:

| SLI | Example |
|---|---|
| **Availability** | Percentage of requests that return a non-5xx response |
| **Latency** | P99 response time for API requests |
| **Error rate** | Percentage of requests resulting in an error |
| **Throughput** | Requests per second the service can sustain |
| **Durability** | Percentage of data written that can be successfully read back |
| **Freshness** | Time since data was last updated (for data pipelines) |

SLIs should be:
- **Meaningful**: directly tied to user experience
- **Measurable**: collected via metrics infrastructure
- **Normalized**: expressed as a ratio (0–100%) where possible, to make target-setting intuitive

### SLO — Service Level Objective

An SLO is an internal target that defines the threshold at which an SLI is considered acceptable:

```
SLI: Availability = (successful requests / total requests) × 100%
SLO: Availability ≥ 99.9% over a rolling 30-day window
```

SLOs are **internal** agreements — they guide engineering decisions, alert thresholds, and on-call responses. They are typically stricter than the SLA to leave a buffer.

**Error budget**: The difference between 100% and the SLO target is the **error budget** — the acceptable amount of "badness" per period.

```
SLO: 99.9% availability over 30 days
Total minutes in 30 days: 43,200
Error budget: 0.1% × 43,200 = 43.2 minutes of allowable downtime/errors
```

Error budgets make reliability trade-offs concrete:
- If the error budget is nearly exhausted, slow down deployments and focus on reliability.
- If the error budget has plenty of room, move fast and ship features.
- This aligns incentives: SREs and product teams share a budget, avoiding conflict between "move fast" and "stay reliable."

### SLA — Service Level Agreement

An SLA is a contract with external customers that defines the minimum service level and the consequences if it is not met. Consequences typically include:
- **Service credits**: customers receive partial refunds or credits against future bills.
- **Penalty clauses**: financial penalties paid by the provider.
- **Termination rights**: customers may exit contracts without penalty if SLA is breached.

SLAs are typically:
- **Looser than SLOs**: you promise customers 99.9%, but your internal target is 99.95% — the buffer absorbs minor incidents without affecting the SLA.
- **Negotiated**: enterprise customers often negotiate custom SLAs.
- **Legally reviewed**: unlike SLOs, SLAs are legal documents.

**Example (typical cloud SLA structure):**
```
SLA: 99.9% monthly uptime
Breach 99.0–99.9%: 10% service credit
Breach 95.0–99.0%: 25% service credit
Breach < 95.0%: 100% service credit
```

### The Nines

| Availability | Downtime per month | Downtime per year |
|---|---|---|
| 99% (2 nines) | 7.3 hours | 3.65 days |
| 99.9% (3 nines) | 43.8 minutes | 8.76 hours |
| 99.95% | 21.9 minutes | 4.38 hours |
| 99.99% (4 nines) | 4.4 minutes | 52.6 minutes |
| 99.999% (5 nines) | 26 seconds | 5.26 minutes |

Each additional nine requires roughly 10× more engineering investment. Most production services target 3–4 nines.

### Setting Practical SLOs

1. **Measure current SLIs** — you can't set targets without historical data.
2. **Identify user pain thresholds** — what level of latency or error rate do users notice and complain about?
3. **Account for dependencies** — your SLO can never be better than the weakest dependency's SLO (see availability in series: $A_1 \times A_2 \times \cdots$).
4. **Start conservatively** — set an SLO you're confident you can achieve, then tighten it incrementally.
5. **Review and adjust** — SLOs should evolve as the system matures.

---

## Key Trade-offs

| | More aggressive SLO | More relaxed SLO |
|---|---|---|
| User experience | Better | Worse |
| Engineering cost | Higher | Lower |
| Error budget | Smaller (less room to fail) | Larger (more room to experiment) |
| Deployment velocity | Slower | Faster |

---

## When to apply

- **SLIs and SLOs** apply to every service in production — even internal services. If users depend on it, you should be measuring it.
- **SLAs** are needed when external customers have contractual expectations of reliability.
- **Error budgets** are most valuable in organizations where reliability and velocity are in tension — they make the trade-off explicit and data-driven.

---

## Common Pitfalls

- **SLO too strict**: A 99.999% SLO that the team can't realistically achieve leads to constant breaches, on-call fatigue, and paralysis.
- **SLO too loose**: A 99% SLO allows 7+ hours of monthly downtime — unacceptable for most user-facing services.
- **Measuring availability without measuring latency**: A service that returns 200 OK in 30 seconds is "available" but unusable. Always include latency SLIs alongside availability.
- **No alerting on error budget burn rate**: Waiting for the SLO to breach before acting is too late. Alert when the burn rate is high enough to exhaust the budget ahead of schedule.
- **Setting SLOs without measuring dependencies**: If your database is 99.9% available and your cache is 99.9% available, your service as a whole cannot guarantee better than $0.999 \times 0.999 \approx 99.8\%$ availability.

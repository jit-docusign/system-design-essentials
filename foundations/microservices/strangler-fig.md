# Strangler Fig Pattern

## What it is

The **Strangler Fig pattern** is an approach to incrementally migrating a monolithic application to microservices (or a new architecture) by gradually extracting functionality piece-by-piece — without rewriting the entire system at once.

The name comes from a plant that grows around a host tree, slowly replacing it until the original is gone. Similarly, new services grow around the monolith, intercept its traffic, and eventually replace it in full while the monolith continues to run throughout the migration.

This is the safest, most practical strategy for modernizing large legacy systems with zero/minimal downtime risk.

---

## How it works

### The Three Phases

**Phase 1: Identify and Extract**  
Choose a bounded context or feature (e.g., User Authentication) that can be meaningfully extracted as a standalone service. It should be:
- Relatively isolated (few outbound dependencies on the monolith).
- High-value to extract (gets a lot of development changes, or is a bottleneck).
- Small enough to extract in a short sprint cycle.

**Phase 2: Route Traffic via a Facade**  
Introduce a façade (reverse proxy, API gateway, or router) in front of the monolith. Route requests for the extracted functionality to the new service; all other requests go to the monolith.

```
Before:
  Client ──► Monolith (handles everything)

During migration:
  Client ──► Router (Façade)
                ├── /users/*, /auth/* ──► New User Service (microservice)
                └── everything else  ──► Monolith (legacy)

After full migration:
  Client ──► Router ──► [only microservices, monolith retired]
```

**Phase 3: Expand and Retire**  
Iteratively extract more bounded contexts. Over time, the monolith shrinks as more functionality is moved to services. When the monolith is empty (or too small to matter), decommission it.

### Routing Façade Options

| Option | Description | Best for |
|---|---|---|
| **API Gateway** | Kong, AWS API Gateway — route by path or header | REST APIs, external traffic |
| **Reverse proxy** | Nginx, Envoy — URL rewrite + proxy | HTTP traffic, internal or external |
| **Branch by abstraction** | Feature flag in monolith code | In-process routing, no network hop |
| **Event interception** | Mirror events to both monolith + service | Async, event-driven architectures |

### Data Migration Strategy

The hardest part of the strangler fig is data — how do you move data from the monolith's database to the new service's database while both are running?

**Step 1: Strangler with shared database**
Initially, the new service reads/writes to the same database as the monolith. Fast to get started but creates coupling.

```
New User Service ──► Shared Monolith DB (users table) 
Monolith         ──► Shared Monolith DB (users table)
```

**Step 2: Synchronize databases in parallel**
Run the new service's own database alongside the monolith DB. Write to both; read from the old (monolith) as the source of truth.

```
New User Service ──► New User DB (write)
                 ──► Monolith DB (write + read source of truth)
Monolith         ──► Monolith DB
```

**Step 3: Switch read source**
Once the new DB is consistent, switch reads to the new database. Monolith DB is now the backup.

**Step 4: Remove old writes**
Stop writing to the monolith DB. The monolith accesses the new service via API if it still needs user data.

**Step 5: Retire old data path**
Remove the users table from the monolith DB schema.

### Branch by Abstraction (In-Process Strangling)

Sometimes you can strangle within the monolith itself before extracting the network boundary:

```python
# Phase 1: Original monolith code
def create_user(email, password):
    db.execute("INSERT INTO users ...")
    return user

# Phase 2: Introduce abstraction + feature flag
class UserRepository:
    def create_user(self, email, password):
        if feature_flags.is_enabled('use_new_user_service'):
            return user_service_client.create_user(email, password)
        return self._legacy_create_user(email, password)  # old code

# Phase 3: All traffic to new service → remove old code
```

### Migration Timeline Example

```
Week 1-2:    Extract User Authentication service (highest value, most isolated)
Week 3-4:    Extract Product Catalog service
Week 5-6:    Extract Order Service
Week 7-8:    Extract Payment Service
...
Month 6:     Monolith handles only legacy edge cases
Month 9:     Monolith decommissioned
```

---

## Key Trade-offs

| Advantage | Disadvantage |
|---|---|
| No big-bang rewrite risk (system stays live throughout) | Migration takes months to years — requires sustained commitment |
| Each extraction can be tested and validated independently | The façade/router adds latency and a component to maintain |
| Rollback is easy — re-route traffic back to monolith | Dual-write data strategies are complex and error-prone |
| Business value delivered incrementally | Technical debt of running two systems in parallel |
| Team can learn microservices patterns gradually | Distributed monolith risk if boundaries are drawn incorrectly |

---

## When to use

The Strangler Fig pattern is the right choice when:
- You have a **large, tightly coupled monolith** that can't be rewritten in a "big bang".
- The business needs **continuous delivery** — downtime for a full rewrite is unacceptable.
- You want to **incrementally prove microservices** before going all-in.
- You have **specific bottlenecks** (e.g., auth or product catalog) that need to scale independently.

Alternatives:
- **Modular monolith first**: refactor into well-defined internal modules before extracting services. Often underrated.
- **Big bang rewrite**: only appropriate for small systems or when the monolith is genuinely unreachable and business risk is understood.

---

## Common Pitfalls

- **Extracting the wrong service first**: don't start with the most complex, most-coupled part of the monolith. Start with a small, relatively isolated bounded context to build confidence and tooling.
- **Creating a distributed monolith**: if the new service calls back into the monolith for every request, you've gained nothing. Each extracted service should be self-sufficient — it must own its data.
- **Skipping the data migration plan**: extracting the service boundary without a plan for the database leads to the new service remaining dependent on the monolith DB indefinitely.
- **Not removing old code**: after extracting a service, the old code path in the monolith must be deleted. "Leaving it for now" leads to two sources of truth and divergence.
- **No feature flags for the façade**: without feature flags, testing the new service in production before full rollout is risky. Use flags to enable the new path for a small percentage of traffic first.

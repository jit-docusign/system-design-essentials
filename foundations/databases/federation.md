# Federation (Functional Partitioning)

## What it is

**Federation** (also called **functional partitioning** or **vertical sharding**) is the practice of splitting a **monolithic database into multiple smaller databases organized by functional domain**. Instead of a single database storing all tables (users, products, orders, inventory, etc.), each functional area gets its own dedicated database instance.

Federation is a step toward microservice-level data isolation — a prerequisite for true service independence.

---

## How it works

### The Problem Federation Solves

As an application grows, a monolithic database becomes a bottleneck:
- All services compete for the same connection pool.
- A slow analytics query on the `reports` table degrades checkout latency.
- A schema migration for one team risks impacting unrelated tables.
- The database's CPU/memory has to handle all domains simultaneously.

### Splitting by Functional Domain

```
Before Federation:
┌─────────────────────────────────────┐
│         monolith-db                 │
│  users | orders | products |        │
│  inventory | payments | reviews |   │
│  sessions | notifications | ...     │
└─────────────────────────────────────┘

After Federation:
┌─────────────┐ ┌─────────────┐ ┌─────────────┐
│  users-db   │ │  orders-db  │ │ products-db │
│  users      │ │  orders     │ │  products   │
│  sessions   │ │  payments   │ │  inventory  │
│  profiles   │ │  shipments  │ │  categories │
└─────────────┘ └─────────────┘ └─────────────┘
         ┌─────────────────┐
         │   reviews-db    │
         │   reviews       │
         │   ratings       │
         └─────────────────┘
```

Each database is sized, scaled, and operated independently. An outage or migration in `orders-db` doesn't affect `users-db`.

### Application-Level Data Stitching

Since data is now across multiple databases, joins that were trivial in SQL must be done in application code:

**Before (SQL join across all tables in one DB):**
```sql
SELECT u.name, o.total, p.title
FROM users u
JOIN orders o ON o.user_id = u.id
JOIN order_items oi ON oi.order_id = o.id
JOIN products p ON p.id = oi.product_id
WHERE u.id = 42;
```

**After (federated — multiple queries + application join):**
```python
user = users_db.query("SELECT name FROM users WHERE id = 42")
orders = orders_db.query("SELECT id, total FROM orders WHERE user_id = 42")
order_item_ids = [o['id'] for o in orders]
items = orders_db.query("SELECT * FROM order_items WHERE order_id IN (?)", order_item_ids)
product_ids = [i['product_id'] for i in items]
products = products_db.query("SELECT id, title FROM products WHERE id IN (?)", product_ids)

# merge in application layer
```

This is the classic "N+1 in distributed systems" problem. Mitigations:
- Batch API calls (one call for all needed IDs).
- Cache frequently accessed reference data (product catalog, user profiles).
- Denormalize frequently-joined fields locally (store `user_name` in `orders-db` alongside `user_id`).

### Federation and Microservices

Federation is the data model that makes microservices viable. Each service owns its database — no two services share a database. This is the **Database per Service** pattern.

```
User Service    ─────► users-db    (PostgreSQL)
Order Service   ─────► orders-db   (PostgreSQL)
Product Service ─────► products-db (PostgreSQL)
Search Service  ─────► search-db   (Elasticsearch)
Session Service ─────► sessions-db (Redis)
```

The connection between services is through **APIs**, not the database. This enforces clean boundaries — one service cannot bypass another service's business logic by querying its database directly.

### Handling Cross-Domain Queries

Federation eliminates in-database cross-domain queries. Common strategies:

**1. API composition**: the calling service makes API calls to each needed service and assembles the response. Works well for single-record lookups.

**2. Event-driven denormalization**: each service publishes events when its data changes. Other services consume events and maintain local denormalized copies of the data they need.
```
OrderService publishes: OrderCreated { userId, userEmail, orderId, amount }
ShippingService consumes and stores: userId, userEmail locally
```

**3. CQRS + Event Sourcing**: a read model (query database) aggregates events from multiple services into a cross-domain projection optimized for reporting.

**4. Data warehouse / data lake**: a dedicated analytics store (BigQuery, Redshift, Snowflake) ingests from all service databases via CDC (Change Data Capture) and supports cross-domain analytical queries.

---

## Key Trade-offs

| Advantage | Disadvantage |
|---|---|
| Independent scaling per domain | Cross-domain joins must be handled in application code |
| Fault isolation — one DB down doesn't affect others | Distributed transactions become complex |
| Independent schema evolution per team | Risk of data inconsistency across services |
| Smaller, faster databases per domain | More infrastructure to operate and monitor |
| Supports microservice independence | Cross-domain analytics requires data warehouse |

---

## When to use

Federation is appropriate when:
- Your monolithic database is a **performance or operational bottleneck** and read replicas are insufficient.
- You're **splitting a monolith into microservices** and need data isolation.
- Multiple teams own different domains and need to **evolve schemas independently** without coordination risk.
- Different domains have different **storage and consistency requirements** (e.g., products in relational DB, sessions in Redis, search content in Elasticsearch).

Do not federate prematurely. A single well-tuned database with proper indexing and read replicas handles significant scale. Federation adds operational overhead — apply it when domain boundaries are stable and the scaling justification is clear.

---

## Common Pitfalls

- **Distributed transactions**: actions that span multiple federated databases can't use simple ACID transactions. Use the Saga pattern (sequence of local transactions with compensating transactions on failure) or accept eventual consistency.
- **Chatty cross-service calls**: naive application-level joining leads to many small round-trip API calls. Batch, cache, or denormalize to avoid N+1 patterns.
- **Splitting on wrong domain boundaries**: if two services frequently need each other's data, they may be the wrong split. Revisit the domain model before federating.
- **Missing global IDs**: without careful ID generation, IDs may collide across databases. Use UUIDs or a centralized ID generator.
- **Reporting becomes painful**: cross-domain reports (e.g., revenue by user cohort) require a data warehouse or complex API aggregation. Plan for this before federating.

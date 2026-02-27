# Denormalization

## What it is

**Denormalization** is the deliberate act of introducing **redundancy** into a database schema — storing the same piece of data in more than one place — to optimize **read performance** at the cost of increased **write complexity** and storage.

It is the intentional reversal of normalization. While normalization eliminates redundancy to avoid update anomalies, denormalization accepts that redundancy in exchange for faster, simpler reads.

---

## How it works

### Core Mechanism

In a normalized schema, data is stored once and retrieved by joining tables. In a denormalized schema, data is pre-computed and co-located where it will be read.

**Normalized (3NF):**
```sql
-- users: id, name, email
-- orders: id, user_id, created_at
-- order_items: id, order_id, product_id, quantity, unit_price
-- products: id, name, price

-- To show an order with user name and product names:
SELECT u.name, p.name, oi.quantity, oi.unit_price
FROM orders o
JOIN users u ON u.id = o.user_id
JOIN order_items oi ON oi.order_id = o.id
JOIN products p ON p.id = oi.product_id
WHERE o.id = 12345;
```

**Denormalized (pre-joined for order display):**
```sql
-- order_details: id, user_id, user_name, product_id, product_name,
--                quantity, unit_price, order_created_at
-- Stored redundantly; no join needed for order display page

SELECT user_name, product_name, quantity, unit_price
FROM order_details
WHERE order_id = 12345;
-- Single table scan, no joins
```

### Common Denormalization Techniques

#### 1. Duplicating Columns Across Tables

Store frequently-read foreign key data alongside the FK itself:
```sql
-- Instead of joining users to get the name every time:
orders: id, user_id, user_name, user_email, ...
```
Writes must now update `user_name` in both `users` and `orders` when a user changes their name.

#### 2. Pre-Computing Aggregates

Store computed aggregate values as a column rather than calculating on every read:
```sql
-- Instead of: SELECT COUNT(*) FROM followers WHERE user_id = X
-- Store: users.follower_count (updated on follow/unfollow writes)

-- Instead of: SELECT SUM(quantity) FROM order_items WHERE order_id = X
-- Store: orders.item_count (updated as items are added)
```

This trades write-time computation for read-time computation.

#### 3. Materialized Views

A **materialized view** is a stored query result that is refreshed periodically or on trigger. Unlike a regular view (executed fresh every time), a materialized view pre-computes and stores the result.

```sql
-- PostgreSQL materialized view:
CREATE MATERIALIZED VIEW product_stats AS
  SELECT
    p.id,
    p.name,
    COUNT(r.id) AS review_count,
    AVG(r.rating) AS avg_rating,
    SUM(oi.quantity) AS total_sold
  FROM products p
  LEFT JOIN reviews r ON r.product_id = p.id
  LEFT JOIN order_items oi ON oi.product_id = p.id
  GROUP BY p.id, p.name;

-- Refresh manually or via cron:
REFRESH MATERIALIZED VIEW product_stats;
```

Cost: the materialized view is stale between refreshes. Full refresh on large datasets can be expensive.

#### 4. Embedded / Snowflake Documents

In document databases (MongoDB, DynamoDB), embed related data directly in the parent document:
```json
{
  "_id": "order-123",
  "user": {
    "id": "user-42",
    "name": "Alice",
    "email": "alice@example.com"
  },
  "items": [
    { "product_id": "p-1", "name": "Laptop", "qty": 1, "price": 999.00 },
    { "product_id": "p-2", "name": "Mouse",  "qty": 2, "price": 29.99 }
  ],
  "total": 1058.98
}
```
The entire order is readable in one document fetch. Update cost: updating a product name requires updating every order that referenced it.

#### 5. Derived Summary Tables

Create separate summary tables for analytics:
```sql
-- daily_revenue: computed every night from orders
CREATE TABLE daily_revenue (
  date DATE PRIMARY KEY,
  total_revenue DECIMAL(15, 2),
  order_count INT
);
```

Analytical dashboards query the summary table instantly; the expensive aggregation runs once offline.

### OLTP vs OLAP — A Key Pattern

Normalization is optimized for **OLTP** (transactional, many small reads/writes, low latency, strong consistency). Denormalization is often applied for **OLAP** (analytical, few large reads, aggregations, reporting).

Common pattern: keep the source of truth normalized for writes (OLTP), then ETL/stream to a denormalized analytical store (data warehouse) for reads.

---

## Key Trade-offs

| Gain | Cost |
|---|---|
| Faster reads (fewer or no joins) | More complex write logic — updates must propagate to all copies |
| Simpler read queries | Risk of data inconsistency if one copy is missed |
| Reduced read latency at scale | More storage consumed |
| Avoids expensive join operations | Harder to enforce referential integrity |
| Good for read-heavy workloads | Schema is harder to understand and maintain |

---

## When to use

- **Read-heavy workloads**: the page/query is read thousands of times per write.
- **Join-heavy queries that dominate latency**: the join is proven to be the performance bottleneck via profiling.
- **Analytics and reporting**: separate analytic store from transactional store; report off the denormalized store.
- **Pre-computed feed generation**: social feeds (user timelines) are often pre-generated and stored denormalized to avoid costly fan-out queries at read time.
- **Document model use cases**: when data is always read together, embedding it in one document is natural.

---

## Common Pitfalls

- **Denormalizing without measuring**: profiling should confirm that joins are the bottleneck before adding redundancy. Don't denormalize speculatively.
- **Stale data inconsistency**: if a product name is stored redundantly in 10 places, an update to the product name must update all 10 — missing even one location creates inconsistency. Use transactions or events to keep copies synchronized.
- **Not considering write amplification**: denormalization multiplies write work. An application that is write-heavy may degrade after denormalization because each write now updates many places.
- **Over-denormalization**: storing every possible precomputed value defeats the purpose — data becomes unmanageable. Be selective; denormalize only columns that are provably read-hot.
- **Treating denormalization as the first step**: always normalize first, find the bottleneck, then selectively denormalize with a clear understanding of the consistency trade-off.

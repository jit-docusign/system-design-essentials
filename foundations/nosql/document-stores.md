# Document Stores

## What it is

A **document store** (document database) organizes data as **self-describing, semi-structured documents** — typically JSON or BSON — grouped in **collections** (analogous to tables). Unlike key-value stores, document databases understand the structure of the stored value and allow querying by any field within it.

Documents can have **nested objects and arrays**, and documents within the same collection can have different fields — a flexible schema that suits rapidly evolving data models.

**Common systems**: MongoDB, CouchDB, Amazon DocumentDB, Couchbase, Firestore (Firebase).

---

## How it works

### Data Model

Data is stored as **documents** within **collections**. Each document has a unique `_id` (primary key):

```json
// Collection: users
{
  "_id": "usr-001",
  "name": "Alice Johnson",
  "email": "alice@example.com",
  "address": {
    "city": "San Francisco",
    "state": "CA",
    "zip": "94107"
  },
  "tags": ["premium", "beta-tester"],
  "preferences": {
    "theme": "dark",
    "notifications": true
  },
  "created_at": "2024-01-15T10:00:00Z"
}

// Collection: products (different shape — valid in same DB)
{
  "_id": "prd-890",
  "title": "Mechanical Keyboard",
  "price": 149.99,
  "variants": [
    {"color": "black", "stock": 40},
    {"color": "white", "stock": 15}
  ],
  "specs": {
    "switch_type": "Cherry MX Brown",
    "layout": "TKL"
  }
}
```

A collection imposes no rigid schema — documents can have varying fields as long as they have `_id`.

### Querying

Document databases provide rich query capabilities over document fields, unlike key-value stores:

```javascript
// MongoDB query syntax (JavaScript-style)

// Find by field value
db.users.find({ "address.city": "San Francisco" })

// Query nested field
db.users.find({ "preferences.theme": "dark" })

// Array contains value
db.users.find({ "tags": "premium" })

// Range query
db.products.find({ "price": { $gte: 100, $lte: 200 } })

// Text search
db.products.find({ $text: { $search: "mechanical keyboard" } })

// Projection — return only specific fields
db.users.find({ "tags": "premium" }, { "name": 1, "email": 1 })
```

**Indexes** can be created on any field within a document, including nested fields and array elements.

### Embedding vs Referencing

The most important schema design decision in document databases: should related data be **embedded** in the document, or **referenced** (FK-style link to another collection)?

**Embed** when:
- Data is always read together.
- The nested data belongs to exactly one parent (1:1 or 1:few relationship).
- The nested data doesn't change frequently.

```json
// Embedded — order_items are always read with the order
{
  "_id": "ord-123",
  "user_id": "usr-001",
  "status": "shipped",
  "items": [
    { "product_id": "prd-890", "name": "Keyboard", "qty": 1, "price": 149.99 },
    { "product_id": "prd-891", "name": "Mouse Pad", "qty": 2, "price": 19.99 }
  ],
  "total": 189.97
}
```

**Reference** when:
- Data is shared across many documents (e.g., a product referenced in thousands of orders).
- The referenced data changes frequently (updating one document updates all users of it).
- The embedded array could grow unboundedly.

```json
// Referenced — product data is shared and may change
{
  "_id": "ord-124",
  "user_id": "usr-001",
  "items": [
    { "product_id": "prd-890", "qty": 1 }  // reference only
  ]
}
// Fetch product details separately when needed
```

Unlike SQL FKs, document database references are not enforced by the DB — referential integrity is the application's responsibility.

### Aggregation Pipeline

MongoDB's aggregation pipeline processes documents through a sequence of transformation stages:

```javascript
db.orders.aggregate([
  { $match: { status: "completed", created_at: { $gte: ISODate("2024-01-01") } } },
  { $unwind: "$items" },
  { $group: {
      _id: "$items.product_id",
      total_revenue: { $sum: { $multiply: ["$items.price", "$items.qty"] } },
      units_sold: { $sum: "$items.qty" }
  }},
  { $sort: { total_revenue: -1 } },
  { $limit: 10 }
])
```

This is equivalent to a SQL GROUP BY + JOIN — but operates on embedded array elements.

### Transactions

Early MongoDB versions (< 4.0) only offered single-document atomicity. Multi-document transactions (across multiple documents or collections) were added in MongoDB 4.0+, supporting full ACID semantics — but with performance overhead. Single-document operations remain the primary consistency unit.

**Design for atomicity**: store data that must change atomically in the same document. This is the document modeling principle that avoids the need for multi-document transactions.

### Horizontal Scaling

MongoDB Sharding:
- Collections are sharded by a **shard key** (e.g., `user_id`, `tenant_id`).
- Data is partitioned into **chunks** and distributed across shard nodes.
- A **mongos** router directs queries to the correct shard(s).

Changes to shard key values require migrating documents — choose an immutable, high-cardinality shard key.

---

## Key Trade-offs

| Advantage | Disadvantage |
|---|---|
| Flexible schema — evolve fields without migrations | No joins — cross-collection queries are expensive |
| Natural fit for hierarchical / nested data | No referential integrity enforcement |
| Rich query on any field | Multi-document transactions have overhead |
| Horizontal scaling by design | Schema inconsistency risk without discipline |
| Good for object-relational impedance mismatch | Deep nesting leads to complex queries |

---

## When to use

Document stores are a good fit when:
- Data is naturally **hierarchical** with nested objects and arrays (product variants, CMS content, user profiles).
- The **schema evolves frequently** during development and you need to avoid migration friction.
- **Read patterns align with document boundaries** — you always read the full document (or large portions of it) together.
- You're building **content management systems**, **catalogs**, **user profiles**, or **event logs**.
- You don't require complex cross-document joins for most queries.

---

## Common Pitfalls

- **Unbounded array growth**: embedding a field that grows indefinitely (e.g., all comments on a post embedded in the post document) causes the document to grow without limit, degrading read/write performance. Reference high-growth arrays.
- **No indexes on query fields**: document stores don't scan efficiently without indexes, just like relational databases. Add indexes on all commonly-queried fields.
- **Treating document store as a general-purpose database**: if your data model is naturally relational (many-to-many relationships, complex joins, ACID multi-table), don't force it into a document model.
- **No schema validation**: many teams skip schema validation in document stores and pay for it with inconsistent, unqueryable data. Use JSON Schema Validation (MongoDB) or application-level validation.
- **Large documents with seldom-accessed fields**: if 90% of a document's bytes are rarely read, projection queries become wasteful. Consider separating hot and cold data.

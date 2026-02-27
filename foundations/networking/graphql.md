# GraphQL

## What it is

**GraphQL** is a query language for APIs and a runtime for executing those queries, developed by Facebook (Meta) in 2012 and open-sourced in 2015. Instead of a server defining fixed endpoints that return fixed data shapes, GraphQL lets **clients declare exactly what data they need** — no more, no less.

This solves two pervasive REST problems:
- **Over-fetching**: the API returns more data than the client needs.
- **Under-fetching**: the client needs to make multiple API calls to assemble the data it needs.

---

## How it works

### Core Concepts

#### Schema

Every GraphQL API is defined by a **schema** — a type system that declares all the types, fields, queries, mutations, and subscriptions the API supports:

```graphql
type User {
  id: ID!
  name: String!
  email: String!
  orders: [Order!]!
}

type Order {
  id: ID!
  total: Float!
  status: String!
  items: [OrderItem!]!
}

type Query {
  user(id: ID!): User
  users(limit: Int, offset: Int): [User!]!
}

type Mutation {
  createUser(name: String!, email: String!): User!
  updateUserName(id: ID!, name: String!): User!
}

type Subscription {
  orderStatusChanged(orderId: ID!): Order!
}
```

The schema is the contract between client and server. It is strongly typed and introspective — clients can query the schema itself to discover what's available.

#### Query

Clients send a query specifying exactly the fields they want:

```graphql
query GetUserWithOrders {
  user(id: "42") {
    name
    email
    orders {
      id
      total
      status
    }
  }
}
```

Response:
```json
{
  "data": {
    "user": {
      "name": "Alice",
      "email": "alice@example.com",
      "orders": [
        { "id": "99", "total": 59.99, "status": "shipped" }
      ]
    }
  }
}
```

The client gets only the fields it asked for — no excess. And it assembled data from a user and their orders in **one request** — no multiple round trips.

#### Mutation

Mutations are used for write operations (create, update, delete):

```graphql
mutation UpdateName {
  updateUserName(id: "42", name: "Alicia") {
    id
    name
  }
}
```

#### Subscription

Subscriptions provide real-time updates over a WebSocket:

```graphql
subscription WatchOrder {
  orderStatusChanged(orderId: "99") {
    status
    updatedAt
  }
}
```

The server pushes events to the client whenever the order status changes.

### Resolvers

On the server side, each field in the schema is backed by a **resolver** — a function that knows how to fetch or compute the value for that field:

```javascript
const resolvers = {
  Query: {
    user: (_, { id }) => db.users.findById(id),
  },
  User: {
    orders: (user) => db.orders.findByUserId(user.id),
  },
};
```

### The N+1 Problem

The resolver per-field model creates a classic problem: if you query 100 users and each user resolver independently fetches that user's orders:

```
1 query to get 100 users → for each user, 1 query to get orders = 100 + 1 = 101 queries
```

This is the **N+1 problem**. The standard solution is a **DataLoader** — a batching and caching utility that:
1. Collects all individual requests within a single execution tick.
2. Sends one batched query to the database for all IDs at once.
3. Returns results to each individual resolver.

### Introspection

GraphQL APIs are **self-describing**. Clients can query the schema itself:

```graphql
{
  __schema {
    types {
      name
      fields { name type { name } }
    }
  }
}
```

This powers tools like GraphiQL (interactive IDE), Apollo Studio, and automatic documentation generation.

---

## GraphQL vs REST vs gRPC

| | REST | GraphQL | gRPC |
|---|---|---|---|
| **Over-fetching** | Common | Eliminated | Not applicable |
| **Under-fetching** | Common (multiple round trips) | Eliminated | Not applicable |
| **Schema** | Optional | Required | Required (.proto) |
| **Versioning** | URL or header versioning | Deprecate fields, rarely version | Field number discipline |
| **Performance** | Good | Good (can have N+1) | Best |
| **Caching** | HTTP caching works naturally | Harder (all POSTs) | Not applicable |
| **Type safety** | Optional | Built-in | Built-in |
| **Browser support** | Native | Native (over HTTP) | Needs proxy |
| **Best for** | Public CRUD APIs | Complex, client-driven queries | Internal services |

---

## Key Trade-offs

| Trade-off | Description |
|---|---|
| **Flexibility vs complexity** | Clients can query anything — but this requires more sophisticated server-side code and N+1 handling |
| **Caching difficulty** | REST uses URL-based HTTP caching naturally; GraphQL queries are typically POSTs, making HTTP-level caching hard — requires application-level caching |
| **Query depth / complexity attacks** | A malicious client can send deeply nested queries that are expensive to execute. Require query depth limiting and cost analysis |
| **Over-exposure of data model** | The schema exposes your entire data model to all clients. Security requires field-level authorization |

---

## When to use

**Use GraphQL when:**
- Multiple clients (web, mobile, partner) have different data needs from the same API.
- Clients are front-end teams that want control over what data they fetch.
- Your data is highly relational and clients frequently need to join data from multiple resources.
- Rapid product iteration — clients can add fields to queries without waiting for backend changes.

**Avoid GraphQL when:**
- Simple CRUD APIs with uniform access patterns — REST is simpler.
- High cacheability of responses matters — REST with HTTP caching is more natural.
- Public APIs for external developers who expect familiar REST conventions.
- You're building internal service-to-service APIs — gRPC is more efficient.

---

## Common Pitfalls

- **Not solving N+1**: GraphQL without DataLoader will fire N+1 database queries for any list query involving nested types. Always implement batching for list resolvers.
- **No query complexity limits**: A deeply nested or wide query can time out your database. Implement max query depth (e.g., 10 levels) and cost-based query analysis.
- **Exposing internal data model directly**: The GraphQL schema becomes a surface area for clients. Think carefully about what fields to expose; apply field-level authorization.
- **Treating subscriptions as simple**: WebSocket-based subscriptions require careful connection management, backpressure handling, and fan-out architecture — they're more complex than queries/mutations.
- **No persisted queries for production**: Allow only pre-approved (persisted) queries in production to prevent arbitrary expensive queries from external clients.

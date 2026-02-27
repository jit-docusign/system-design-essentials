# API Contracts

## What it is

An **API contract** is a formal, explicit agreement between a service (provider) and its callers (consumers) about the shape, behavior, and semantics of the API — fields, types, error codes, versioning, and compatibility rules.

In microservices architectures, services are developed by independent teams and deployed independently. Without enforced contracts, a provider changing their API breaks consumers — sometimes silently. API contracts make these agreements machine-verifiable, catching breaking changes before they reach production.

Common approaches: **OpenAPI/Swagger** (REST), **Protocol Buffers** (gRPC), **AsyncAPI** (event-driven), **Consumer-Driven Contract Testing** (Pact), **JSON Schema**.

---

## How it works

### Contract Types

**Provider contract**: the provider defines the API and publishes a specification. Consumers must conform to it.

```yaml
# OpenAPI 3.0 — Provider defines the contract
openapi: 3.0.0
info:
  title: Order Service API
  version: 2.1.0
paths:
  /orders:
    post:
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateOrderRequest'
      responses:
        '201':
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Order'

components:
  schemas:
    CreateOrderRequest:
      type: object
      required: [user_id, items]
      properties:
        user_id:
          type: string
          example: usr-123
        items:
          type: array
          items:
            $ref: '#/components/schemas/OrderItem'
```

**Consumer-driven contract (CDC)**: consumers define what they expect from the provider. The provider must satisfy all consumer contracts. This shifts the accountability — providers can't change fields that any consumer depends on.

```
Consumer A (Analytics): expects { order_id, user_id, total, timestamp }
Consumer B (Notification): expects { order_id, user_id, email }

Provider must return at minimum all fields any consumer depends on.
If provider wants to rename "total" to "amount" → Contract tests FAIL → blocked.
```

### Protocol Buffers (gRPC)

gRPC uses **`.proto` files** as the contract. Strongly typed, language-independent, binary serialized.

```protobuf
// order_service.proto — the contract
syntax = "proto3";

package order;

service OrderService {
  rpc CreateOrder (CreateOrderRequest) returns (Order);
  rpc GetOrder    (GetOrderRequest)    returns (Order);
}

message CreateOrderRequest {
  string user_id = 1;
  repeated OrderItem items = 2;
}

message Order {
  string order_id = 1;
  string user_id  = 2;
  repeated OrderItem items = 3;
  double total    = 4;
  string status   = 5;
  int64  created_at = 6;
}

message OrderItem {
  string product_id = 1;
  int32  quantity   = 2;
  double price      = 3;
}
```

The `.proto` file is checked into source control, distributed to all consumers, and used to generate client/server code. Consumers compile against the proto — if the provider removes a field, compilation fails.

**Schema evolution in Protobuf:**
- **Adding** new fields: safe — old consumers ignore unknown fields.
- **Renaming** fields: safe — field names aren't in the binary, only numbers.
- **Removing** fields: use `reserved` to prevent field number reuse.
- **Changing field types incompatibly**: breaking.

```protobuf
message Order {
  reserved 7, 8;          // previously used field numbers, never reuse
  reserved "old_field";   // previously used field names
}
```

### Consumer-Driven Contract Testing (Pact)

**Pact** is a framework for CDC testing. Consumers write tests specifying what they expect from providers; Pact records these as **"pact files"** (contracts). Providers run their test suite against the generated pact files to verify they satisfy all consumers.

```python
# Consumer (notification-service) defines its expectation:
from pact import Consumer, Provider

pact = Consumer('notification-service').has_pact_with(Provider('order-service'))

pact.given('an order exists') \
    .upon_receiving('a request for order details') \
    .with_request(method='GET', path='/orders/order-123') \
    .will_respond_with(
        status=200,
        body={
            'order_id': 'order-123',
            'user_id': 'usr-456',
            'email': 'user@example.com'
        }
    )
```

```python
# Provider (order-service) verifies it satisfies all consumers:
from pact.verifier import Verifier

verifier = Verifier(provider='order-service',
                    provider_base_url='http://localhost:8080')
verifier.verify_with_broker(broker_url='https://pact-broker.internal/')
# Fails if the response doesn't match any consumer's pact file
```

Workflow:
```
Consumer writes pact test
  → pact file published to Pact Broker
     → Provider CI pipeline fetches pact file + runs verification
        → FAIL if provider breaks consumer expectations
           → Deploy is blocked
```

### API Versioning Strategies

| Strategy | Example | Trade-off |
|---|---|---|
| **URI versioning** | `/v1/orders`, `/v2/orders` | Simple, explicit, easy to route. URL is polluted. |
| **Header versioning** | `Accept: application/vnd.orders.v2+json` | Clean URLs, but harder to test in browser. |
| **Query parameter** | `/orders?version=2` | Simplest, convenient, but mixes data with meta. |
| **Semantic versioning** | `API-Version: 2.1.0` | Expressive; requires server-side version parsing. |

**Backward compatibility rules** (never break these without a version bump):
- Never remove a required field.
- Never change a field type incompatibly.
- Never rename a field that consumers depend on.
- Never change error code semantics.

**Safe changes** (backward-compatible):
- Add optional fields.
- Add new endpoints.
- Add new error codes (consumers should handle unknown codes).

### Schema Registry

For event-driven systems, a **schema registry** (Confluent Schema Registry, AWS Glue Schema Registry) stores Avro/Protobuf/JSON Schema definitions centrally. Producers and consumers validate messages against the registry at runtime.

```
Producer publishes event → Schema Registry validates schema → message sent to Kafka
Consumer reads event → Schema Registry fetches schema → deserialize and validate
```

Enforces schema evolution rules: BACKWARD, FORWARD, FULL compatibility modes.

---

## Key Trade-offs

| Advantage | Disadvantage |
|---|---|
| Breaking changes caught in CI before deployment | Contract tests require buy-in from all teams |
| Self-documenting APIs (OpenAPI / protobuf) | Overhead of maintaining contract files and versioning |
| Multiple language clients generated from one spec | CDC testing adds pipeline complexity |
| Clear compatibility rules reduce integration bugs | Loose contract checking can still miss semantic regressions |

---

## When to use

- **Cross-team API dependencies**: when the provider and consumer are owned by different teams with separate release cycles.
- **Public or partner APIs**: external consumers cannot see your source code; a formal contract is the only spec they have.
- **gRPC services**: protobuf is the contract by default — distribute and version `.proto` files.
- **Event schemas**: use AsyncAPI or a schema registry to govern event-driven interfaces.

---

## Common Pitfalls

- **Implicit contracts**: no spec, no tests — "just check the code." Works until a team unknowingly renames a field. Always make contracts explicit and tested.
- **Version creep**: multiple live API versions forever. Define a sunset policy — old versions deprecated 6 months after a new major version; removed at 12 months.
- **Over-restrictive contracts**: consumer-driven contracts should specify only what the consumer actually needs, not the complete provider response. Otherwise, providers can't evolve freely.
- **Generated code not checked in**: if client SDKs generated from protobuf/OpenAPI aren't committed, different services may build against different versions of the generated code.

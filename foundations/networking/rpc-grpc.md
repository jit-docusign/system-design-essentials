# RPC / gRPC

## What it is

**RPC (Remote Procedure Call)** is a communication paradigm where a program on one machine calls a procedure (function) on another machine as if it were a local function call. The network transport and serialization are hidden behind an interface that looks like a normal function call.

**gRPC** is a modern, high-performance, open-source RPC framework developed by Google, built on **HTTP/2** and using **Protocol Buffers (Protobuf)** as its serialization format. It is the dominant RPC framework for internal service-to-service communication in microservices architectures.

---

## How it works

### Core Idea

```
Without RPC (manual):
  Server A serializes → sends HTTP request → Server B deserializes → processes → serializes → returns → Server A deserializes

With RPC (abstracted):
  Server A calls: getUserById(42) → gets back User{id:42, name:"Alice"}
  (network transport is invisible to the call site)
```

### Protocol Buffers (Protobuf)

Protobuf is a binary serialization format. Service interfaces and message shapes are defined in `.proto` files:

```proto
syntax = "proto3";

service UserService {
  rpc GetUser (GetUserRequest) returns (User);
  rpc ListUsers (ListUsersRequest) returns (stream User);  // server streaming
  rpc UpdateUser (User) returns (User);
}

message GetUserRequest {
  int64 user_id = 1;
}

message User {
  int64 id = 1;
  string name = 2;
  string email = 3;
  repeated string roles = 4;
}
```

From this `.proto` file, **code generators** produce client and server stubs in your language of choice (Go, Java, Python, C++, Kotlin, etc.). The generated code handles serialization, deserialization, and transport entirely.

### Why Protobuf is Faster Than JSON

| Property | JSON | Protobuf |
|---|---|---|
| Format | Text (UTF-8) | Binary |
| Size | Larger (field names repeated in every record) | Smaller (~3–10× compression vs JSON) |
| Parsing | CPU-intensive text parsing | Fast binary decode |
| Schema | None (or separate) | Embedded in .proto definition |
| Human-readable | Yes | No (requires tooling) |

### HTTP/2 Transport

gRPC uses HTTP/2 as its transport, gaining:
- **Multiplexing**: Multiple RPC calls over a single TCP connection (no head-of-line blocking between calls).
- **Header compression**: Headers compressed with HPACK.
- **Bidirectional streaming**: Enables streaming RPCs.
- **Flow control**: Per-stream flow control.

### gRPC Communication Patterns

| Pattern | Description | Example use case |
|---|---|---|
| **Unary** | Client sends one request, server sends one response | `GetUser(id)` → `User` |
| **Server streaming** | Client sends one request, server streams many responses | `WatchEvents(filter)` → `stream Event` |
| **Client streaming** | Client streams many requests, server sends one response | `UploadChunks(stream bytes)` → `FileUploadResult` |
| **Bidirectional streaming** | Both sides stream simultaneously | Real-time bidirectional data sync |

```proto
// Unary
rpc GetUser (GetUserRequest) returns (User);

// Server streaming
rpc ListOrders (ListOrdersRequest) returns (stream Order);

// Client streaming
rpc RecordMetrics (stream MetricPoint) returns (RecordResult);

// Bidirectional streaming
rpc Chat (stream ChatMessage) returns (stream ChatMessage);
```

### Deadlines / Timeouts

gRPC has first-class support for deadlines. A client specifies the maximum time it's willing to wait for the entire RPC to complete — the deadline propagates through the call chain automatically:

```
Client sets deadline: 500ms
  → Service A (100ms used) → Service B (deadline now 400ms left)
    → Service C (deadline now 200ms left)
if Service C doesn't respond in 200ms → context cancelled → all services abort
```

This is critical for preventing cascading timeouts in microservices chains.

### gRPC vs REST Comparison

| | REST | gRPC |
|---|---|---|
| **Protocol** | HTTP/1.1 or HTTP/2 | HTTP/2 |
| **Serialization** | JSON (text) | Protobuf (binary) |
| **Schema** | Optional (OpenAPI) | Required (.proto) |
| **Code generation** | Optional | First-class (client/server stubs) |
| **Browser support** | Native | Needs gRPC-web proxy |
| **Streaming** | Limited (SSE, WebSockets) | Native (4 patterns) |
| **Performance** | Good | Better (binary, mux, compression) |
| **Discoverability** | High (HTTP conventions) | Lower (requires .proto) |

---

## Key Trade-offs

| Trade-off | Description |
|---|---|
| **Performance vs flexibility** | Protobuf is faster but requires schema definition; JSON is slower but more flexible |
| **Strong typing vs loose coupling** | .proto enforces a strict contract; schema changes require versioning discipline |
| **Internal vs external** | gRPC is excellent for internal services; for public APIs, REST or GraphQL are more accessible |
| **Browser compatibility** | gRPC cannot be called natively from browsers; requires gRPC-Web with a proxy |

---

## When to use

**Use gRPC when:**
- Service-to-service communication within a microservices backend.
- Performance is critical (high throughput, low latency internal APIs).
- You want strong schema contracts and auto-generated client/server code.
- Streaming is needed (server push, bidirectional communication).
- Propagating deadlines and metadata automatically across service boundaries matters.

**Use REST when:**
- Building a public API for external consumers.
- Browser clients need to call the API directly (without a proxy).
- Simplicity and human-readability are priorities.
- The team is unfamiliar with Protobuf tooling.

---

## Common Pitfalls

- **Breaking Protobuf schema compatibility**: Adding or removing fields by number can break existing clients. Follow the rules: never reuse field numbers, only add new fields, use `reserved` to mark removed fields.
- **Not setting deadlines**: gRPC calls without deadlines can hang indefinitely if a downstream service is stuck. Always set deadlines.
- **Ignoring gRPC status codes**: gRPC has its own status code system (OK, CANCELLED, DEADLINE_EXCEEDED, NOT_FOUND, etc.) — map application errors to the correct status code for proper client handling.
- **Assuming gRPC is drop-in for public APIs**: gRPC-Web requires a proxy (Envoy, grpc-gateway) and is not natively supported by all clients. For public APIs, REST or GraphQL is more practical.
- **Streaming without backpressure**: Bidirectional streaming without flow control can cause one fast side to overwhelm a slow receiver. gRPC has built-in flow control — don't bypass it.

# WebSockets

## What it is

**WebSockets** provide a full-duplex, persistent communication channel over a single TCP connection between a client and a server. Unlike HTTP, where the client must initiate every exchange, WebSockets allow **both the client and server to send messages to each other at any time**, independently.

This makes WebSockets the standard technology for real-time, bidirectional communication: live chat, collaborative editing, real-time dashboards, multiplayer gaming, and financial data feeds.

---

## How it works

### Upgrade Handshake

WebSocket connections begin as an HTTP request and then **upgrade** to the WebSocket protocol:

```
Client → Server:
  GET /chat HTTP/1.1
  Host: example.com
  Upgrade: websocket
  Connection: Upgrade
  Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
  Sec-WebSocket-Version: 13

Server → Client:
  HTTP/1.1 101 Switching Protocols
  Upgrade: websocket
  Connection: Upgrade
  Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

After the `101 Switching Protocols` response, the HTTP connection is upgraded — the TCP connection is now a WebSocket tunnel and HTTP semantics no longer apply. Both ends can now send **frames** freely.

### Frame-Based Communication

WebSocket data is sent in **frames**:
- A frame has a small header (2–10 bytes) and a payload.
- Frames can be **text** (UTF-8) or **binary** (arbitrary bytes).
- Frames can be **fragmented** — large messages split across multiple frames.
- Control frames handle lifecycle: **Ping/Pong** (heartbeat), **Close** (graceful shutdown).

```
Client ──────▶ {"type": "message", "text": "Hello"} ──────▶ Server
Client ◀────── {"type": "message", "text": "Hi back"} ◀────── Server
Client ◀────── {"type": "user_joined", "user": "Bob"} ◀────── Server
       ↑ server-initiated push — impossible with plain HTTP
```

### Connection Lifecycle

1. **Establishment**: HTTP upgrade handshake.
2. **Open**: Both sides can send frames freely.
3. **Heartbeat (Ping/Pong)**: Either side can send a Ping; the other must respond with a Pong. Used to detect dead connections and prevent proxy timeouts.
4. **Close**: Either side sends a Close frame; the other side acknowledges with a Close frame; the TCP connection is then closed.

### WebSocket over TLS (WSS)

Unencrypted WebSocket uses the `ws://` scheme; encrypted uses `wss://` (WebSocket Secure), which runs over TLS. In production, always use `wss://`.

### Scaling WebSocket Servers

WebSockets create a **long-lived, stateful connection** — a client is connected to a specific server for the duration of the session. This creates challenges for horizontal scaling:

**Problem:** If Client A is connected to Server 1 and Client B is connected to Server 2, and Server 2 wants to send a message to Client A, it can't — it doesn't have that connection.

**Solutions:**

1. **Pub/Sub message bus (most common)**: All WebSocket servers subscribe to a shared channel (Redis Pub/Sub, Kafka). When a message should be broadcast to a user, it's published to the bus; the server holding that user's connection picks it up and delivers it.

```
Server 1 (has client A) ──subscribes──▶ Redis Pub/Sub Channel "user_A"
Server 2 (has client B, wants to message A) ──publishes──▶ Redis Pub/Sub Channel "user_A"
Server 1 receives from Redis → delivers to Client A's WebSocket
```

2. **Sticky sessions**: Route each client consistently to the same server via IP hash or cookie-based routing in the load balancer. Simpler but reduces load-balancing effectiveness.

3. **Dedicated WebSocket gateway**: Use a specialized service (e.g., Socket.IO, Pusher, Ably) that handles connection state; backend services communicate with it via a simple API.

---

## Key Trade-offs

| Trade-off | Description |
|---|---|
| **Full-duplex vs request-response** | WebSockets enable server push but require persistent connections — each connection holds open a file descriptor and memory on the server |
| **Connection overhead** | At scale (1M+ concurrent connections), memory and file descriptor limits become a constraint; proper server tuning is required |
| **Stateful connections vs stateless HTTP** | WebSocket servers can't be scaled out as easily as stateless HTTP servers; requires a message bus for multi-server deployments |
| **Proxy / firewall compatibility** | Some corporate firewalls and older proxies don't support WebSocket upgrades; `wss://` over port 443 is the most compatible |

---

## When to use

Use WebSockets when:
- **Low-latency server-initiated push is required**: live chat, real-time collaboration, live scores/prices, notifications.
- **High-frequency bidirectional messages**: multiplayer gaming, live cursors in collaborative editing.
- **Avoiding polling overhead**: replacing long-polling or short-polling with a persistent connection reduces HTTP overhead significantly.

Prefer alternatives (SSE, HTTP/2 push, polling) when:
- Only the **server** needs to push (not the client) — SSE is simpler and sufficient.
- The update frequency is low — simple polling is easier to implement and debug.
- Connection persistence is operationally complex and not justified by the use case.

---

## Common Pitfalls

- **Not implementing heartbeats**: Connections that are idle but open may be silently closed by NAT gateways, load balancers, or proxies after a timeout. Send periodic Ping/Pong frames to keep connections alive.
- **No reconnection logic on the client**: Network interruptions are inevitable. Client code must detect disconnection and implement exponential-backoff reconnection.
- **Storing connection state in memory on a single server**: If that server restarts, all connected clients lose their state. Persist session state externally (Redis, database).
- **Using WebSockets for low-frequency, low-latency-tolerant updates**: If you're sending one update per minute, HTTP polling or SSE is simpler and operationally cheaper.
- **Not limiting the number of connections per user**: Malicious or buggy clients can open unlimited connections. Enforce per-user and per-IP limits.

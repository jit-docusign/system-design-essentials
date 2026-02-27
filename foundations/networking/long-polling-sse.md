# Long Polling & Server-Sent Events (SSE)

## What it is

**Long polling** and **Server-Sent Events (SSE)** are two techniques for pushing data from a server to a client using standard HTTP — without requiring the full complexity of WebSockets. They are most appropriate when only the **server pushes data to the client** (unidirectional), and the client does not need to stream data back to the server.

| Technique | Direction | Protocol | Connection |
|---|---|---|---|
| Short polling | Client → Server (repeated) | HTTP | New connection per poll |
| Long polling | Client → Server (open) | HTTP | Request held open until event |
| SSE | Server → Client | HTTP | Persistent HTTP response stream |
| WebSockets | Bidirectional | WebSocket (over TCP) | Persistent bidirectional tunnel |

---

## How it works

### Short Polling (for Comparison)

The client sends a request every N seconds regardless of whether there's new data:

```
Client: GET /updates?since=t1    (t=0)
Server: 200 OK — no new events   (immediate response)

Client: GET /updates?since=t1    (t=5s)
Server: 200 OK — no new events   (immediate response)

Client: GET /updates?since=t1    (t=10s)
Server: 200 OK — [new event!]    (immediate response)
```

**Problem**: Wastes resources — most requests return empty responses. Latency is bounded by the polling interval.

### Long Polling

The client sends a request and the **server holds the connection open** until a new event is available or a timeout occurs. When data arrives, the server responds and the client immediately sends a new request.

```
Client: GET /updates?since=t1    (connection opened, server holds it)
                                  (... 8 seconds pass, no event ...)
Server: 200 OK — [new event!]    (responds when event is available)
Client: GET /updates?since=t2    (immediately sends next request)
```

**If no event occurs before the timeout:**
```
Server: 200 OK — [] (empty, after timeout, e.g. 30s)
Client: GET /updates?since=t1    (retries immediately)
```

**Latency**: Near-zero delay between event occurring and client receiving it — as soon as the event happens, the server responds to the held request.

**Resource cost**: Each waiting client holds an open HTTP connection on the server — a thread or file descriptor. Under load (10K+ clients), this consumes significant server resources. Async I/O frameworks (Node.js, async Python) handle this better than thread-per-connection models.

### Server-Sent Events (SSE)

SSE is a **standardized HTTP-based protocol** for unidirectional server-to-client streaming. The client opens a single HTTP connection; the server streams events down it indefinitely.

**Request:**
```
GET /events HTTP/1.1
Accept: text/event-stream
```

**Response (streams indefinitely):**
```
HTTP/1.1 200 OK
Content-Type: text/event-stream
Cache-Control: no-cache

data: {"type": "price_update", "value": 145.23}

data: {"type": "price_update", "value": 145.45}
id: 42
event: alert
data: {"message": "Price threshold breached"}

: this is a comment (keep-alive)
```

**SSE Event Format:**
- `data:` — the event payload (can span multiple lines).
- `event:` — optional event type (client can subscribe to specific types).
- `id:` — optional event ID; client sends the last received ID on reconnect via `Last-Event-ID` header.
- `: ` — comment (used as heartbeat/keep-alive).

**Automatic Reconnection**: The browser's `EventSource` API automatically reconnects if the connection drops, sending `Last-Event-ID` so the server can replay missed events. This is built into the protocol.

```javascript
const source = new EventSource('/events');
source.addEventListener('alert', (e) => {
  console.log(JSON.parse(e.data));
});
source.onerror = () => console.log('Disconnected, will auto-reconnect');
```

### Long Polling vs SSE

| | Long Polling | SSE |
|---|---|---|
| **Connection** | New connection per event | Single persistent connection |
| **Server overhead** | Higher (new req/resp per event) | Lower (one persistent connection) |
| **Browser support** | Universal | Universal (except some legacy IE) |
| **Auto-reconnect** | Must implement manually | Built into `EventSource` API |
| **Event replay on reconnect** | Must implement manually | Built in via `Last-Event-ID` |
| **HTTP/2 compatibility** | Fine | Better — multiplexed streams |
| **Bidirectional** | Can approximate (slow) | No — server → client only |

---

## Key Trade-offs

| | SSE/Long Polling | WebSockets |
|---|---|---|
| **Simplicity** | High (standard HTTP) | Lower (separate protocol) |
| **Bidirectionality** | SSE: No; Long polling: approximated | Yes |
| **Infrastructure** | Works with existing HTTP infra (CDN, LBs, proxies) | May need special proxy config |
| **Message frequency** | Low to medium | Medium to very high |

---

## When to use

**Use Long Polling when:**
- You need server push but SSE is not available (legacy environment, special constraints).
- Occasional updates (order status, job progress) where per-event connection overhead is acceptable.

**Use SSE when:**
- You need server-to-client streaming with low overhead: live feeds, notifications, real-time dashboards, token streaming from LLM APIs.
- You want automatic reconnection and event replay without implementing it yourself.
- The existing infrastructure (CDNs, proxies) better supports HTTP than WebSockets.

**Use WebSockets when:**
- The client also needs to send high-frequency or continuous data to the server.
- Very high message frequency makes setting up a new request per event (long polling) too slow.

---

## Common Pitfalls

- **SSE and HTTP/1.1 connection limits**: Browsers limit HTTP/1.1 connections per domain to ~6. If a page opens multiple SSE connections to the same domain, it will exhaust this limit. HTTP/2 uses multiplexed streams, solving this.
- **Long polling without proper timeout handling**: If the server doesn't respond within a reasonable timeout, the client hangs indefinitely. Always set both client-side request timeouts and server-side hold timeouts.
- **Not implementing heartbeats**: Idle SSE or long-poll connections may be silently dropped by proxies or NATs. Send periodic comments (`: keep-alive`) or ping events to prevent this.
- **Using long polling for high-frequency updates**: If events fire 10 times per second, the overhead of establishing a new request after each response is significant. Switch to SSE or WebSockets.

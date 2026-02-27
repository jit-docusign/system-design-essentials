# Load Balancing Algorithms

## What it is

A **load balancing algorithm** is the strategy a load balancer uses to decide which backend server should receive the next incoming request. The right algorithm depends on the nature of the traffic, the variability in request processing time, and whether session continuity is required.

---

## How it works

### 1. Round Robin

Requests are distributed sequentially across all servers in a cycle:

```
Request 1 → Server A
Request 2 → Server B
Request 3 → Server C
Request 4 → Server A
Request 5 → Server B
...
```

- **Best for**: uniform requests where each takes roughly the same time to process.
- **Problem**: doesn't account for server capacity differences or in-flight request count — a slow request can leave Server A processing 10 requests while Server B finishes quickly and idles.

### 2. Weighted Round Robin

Like round robin, but each server is assigned a weight proportional to its capacity. A server with weight 3 receives 3× the requests of a server with weight 1.

```
Server A: weight 3 → receives 3 of every 6 requests
Server B: weight 2 → receives 2 of every 6 requests
Server C: weight 1 → receives 1 of every 6 requests
```

- **Best for**: heterogeneous server pools where some servers have more CPU/memory.
- **Problem**: still doesn't account for actual current load — a high-weight server could be overloaded with long-running requests.

### 3. Least Connections

Route the next request to the server with the **fewest active connections**:

```
Server A: 10 active connections
Server B: 5 active connections  ← receives next request
Server C: 8 active connections
```

- **Best for**: requests with variable processing times (some fast, some slow). Naturally avoids sending new requests to an already-busy server.
- **Problem**: connection count is a proxy for load, not a direct measure. A server with 5 long-running expensive connections may be more loaded than one with 20 cheap short connections.

### 4. Weighted Least Connections

Combines Least Connections with server weights:

$$\text{Score} = \frac{\text{active connections}}{\text{weight}}$$

The server with the lowest score receives the next request.

### 5. IP Hash (Source IP Affinity)

The client's source IP is hashed to determine which server receives the request. The same client IP always maps to the same server (as long as the server pool doesn't change).

```
hash("10.0.0.1") % 3 = 1 → Server B (always)
hash("10.0.0.2") % 3 = 0 → Server A (always)
```

- **Best for**: stateful applications where session state is stored on the server (although storing state in the server is itself an anti-pattern for scalable systems).
- **Problem**: uneven distribution if many clients share one IP (corporate NAT); server removal reshuffles all assignments.

### 6. Random

Select a backend uniformly at random:

- Statistically converges to even distribution over many requests.
- Simple to implement; no state to maintain.
- Does not consider server health or load.
- Rarely the best choice but used as a baseline in some implementations.

### 7. Least Response Time

Route to the server with the **lowest combination of active connections and response time**:

$$\text{Score} = \text{active connections} \times \text{avg response time}$$

- **Best for**: heterogeneous environments where server response times differ significantly.
- **More sophisticated** than Least Connections but requires tracking rolling average response times per server.

### 8. Consistent Hashing

A distributed hash function maps each request (by a key, typically a URL or user ID) to a position on a **hash ring**. Servers are also placed on the ring. Each request is routed to the nearest server clockwise on the ring.

```
Hash ring (0 to 2³²):

     0
   /   \
 Server A   Server B
   \   /
  Server C
  
request: hash("user_42") → position 180° → nearest clockwise server = Server B
```

**Key properties:**
- Adding or removing a server only affects the keys that were hashed to that server's neighborhood — not all keys are reshuffled.
- **Virtual nodes**: each physical server is assigned multiple positions on the ring (virtual nodes) to ensure even distribution even with a small number of servers.

- **Best for**: distributed caching (Memcached, Redis cluster), where minimizing cache invalidation on cluster changes is critical.
- See also: [Consistent Hashing](../../scalability/consistent-hashing.md) for a deep dive.

### Algorithm Comparison

| Algorithm | Accounts for load | Session affinity | Rebalances on changes | Complexity |
|---|---|---|---|---|
| Round Robin | No | No | N/A | Very low |
| Weighted Round Robin | Partially | No | N/A | Low |
| Least Connections | Yes | No | N/A | Medium |
| IP Hash | No | Yes (IP-based) | Yes (all keys shift) | Low |
| Least Response Time | Yes | No | N/A | Medium-High |
| Consistent Hashing | No | Yes (key-based) | Minimal | High |

---

## Key Trade-offs

- **Simplicity vs accuracy**: Round robin is simple but accurate only when requests are homogeneous. Least connections is more accurate but requires tracking state.
- **Affinity vs distribution**: Algorithms that provide session affinity (IP hash, consistent hashing) sacrifice even distribution.
- **Cache efficiency vs flexibility**: Consistent hashing optimizes for cache hit rates but is more complex to implement and reason about.

---

## When to use each

| Algorithm | Use case |
|---|---|
| **Round Robin** | Homogeneous, short-lived, stateless requests |
| **Weighted Round Robin** | Mixed-capacity server pool |
| **Least Connections** | Variable-length requests (database queries, streaming) |
| **IP Hash** | Stateful legacy servers that can't externalize session state |
| **Least Response Time** | Performance-critical routing with heterogeneous server response times |
| **Consistent Hashing** | Distributed cache routing; minimizing cache invalidation on topology changes |

---

## Common Pitfalls

- **Using Round Robin for variable-length requests**: a few expensive requests on the same server while others are idle leads to severe imbalance; use Least Connections instead.
- **IP Hash breaks behind NAT**: Clients behind a corporate proxy or shared NAT share one IP, causing all their traffic to route to one server.
- **Not using virtual nodes with Consistent Hashing**: Without virtual nodes, a small number of servers leads to uneven hash ring distribution and hotspots.
- **Forgetting health check integration**: All load balancing algorithms must remove unhealthy servers from the rotation. The algorithm choice is irrelevant if the LB keeps routing to failed backends.

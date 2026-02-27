# TCP vs UDP

## What it is

**TCP (Transmission Control Protocol)** and **UDP (User Datagram Protocol)** are the two dominant transport-layer protocols. They both run on top of IP and define how data is sent between hosts — but with fundamentally different trade-offs.

- **TCP**: reliable, ordered, connection-oriented. Guarantees delivery and ordering at the cost of overhead.
- **UDP**: unreliable, unordered, connectionless. Maximizes speed by removing reliability guarantees.

Choosing the right one is a fundamental system design decision for any networked application.

---

## How it works

### TCP

TCP establishes a **connection** before any data is sent, using the **3-way handshake**:

```
Client                  Server
  │── SYN ────────────▶│
  │◀── SYN-ACK ─────────│
  │── ACK ────────────▶│
  │                     │
  │ Connection established — data flows
```

**Guarantees:**
1. **Delivery**: Every segment sent must be acknowledged; unacknowledged segments are retransmitted.
2. **Ordering**: Segments are numbered (sequence numbers); the receiver reassembles them in order regardless of arrival order.
3. **Error detection**: Checksum on every segment; corrupted segments are dropped and retransmitted.
4. **Flow control**: The receiver advertises a window size — how many bytes it can accept. Sender doesn't exceed this.
5. **Congestion control**: TCP detects network congestion (via packet loss or ECN) and reduces send rate to avoid overwhelming the network (slow start, AIMD).

**Connection teardown (4-way):**
```
Client ── FIN ──────▶ Server
Client ◀── ACK ────── Server
Client ◀── FIN ────── Server
Client ── ACK ──────▶ Server
```

**Cost of reliability**: Each segment requires acknowledgment; any segment loss stalls the entire stream until the segment is retransmitted. This is **head-of-line blocking** — a single lost packet blocks all subsequent data in the stream.

### UDP

UDP sends **datagrams** — independent packets — with no handshake, no acknowledgment, and no ordering guarantee.

```
Client ── [Data] ──────▶ Server (delivered? maybe)
Client ── [Data] ──────▶ Server (delivered? maybe)
Client ── [Data] ──────▶ Server (delivered? maybe)
```

**Properties:**
- **No connection**: packets are fired immediately.
- **No acknowledgment**: sender never knows if packets arrived.
- **No ordering**: packets may arrive out of order.
- **No congestion control**: sender can flood the network.
- **No head-of-line blocking**: a lost packet doesn't block others.
- **Lower overhead**: UDP header is 8 bytes vs TCP's 20+ bytes.

**Error detection**: UDP has a checksum (optional in IPv4, required in IPv6) for detecting corruption, but corrupt packets are simply dropped — not retransmitted.

### Comparison

| Property | TCP | UDP |
|---|---|---|
| Connection | Required (3-way handshake) | None |
| Delivery guarantee | Yes (retransmission) | No |
| Ordering | Guaranteed | Not guaranteed |
| Flow control | Yes | No |
| Congestion control | Yes | No |
| Latency overhead | Higher (ACKs, retransmissions) | Lower |
| Head-of-line blocking | Yes | No |
| Header size | 20+ bytes | 8 bytes |
| Use case | Web, file transfer, databases | Gaming, video, DNS, VoIP |

### QUIC — TCP + UDP hybrid

**QUIC** (used in HTTP/3) is a transport protocol built on top of UDP that implements **TCP-like reliability per stream** without global head-of-line blocking:

- Multiplexes multiple streams; a lost packet in Stream 1 doesn't block Stream 2.
- Connection establishment is faster (0-RTT or 1-RTT vs TCP + TLS).
- Built-in TLS 1.3 — encryption is mandatory and baked in.

QUIC is essentially the answer to: "what would TCP look like if we redesigned it today, with security built in and no head-of-line blocking?"

---

## Key Trade-offs

| | TCP | UDP |
|---|---|---|
| **Latency** | Higher (handshake, ACKs) | Lower |
| **Reliability** | Built-in | Application must implement |
| **Ordering** | Built-in | Application must implement or ignore |
| **Network fairness** | Congestion control shares bandwidth | Can be unresponsive to congestion |
| **Packet loss behavior** | Stalls stream | Packet lost, processing continues |

---

## When to use each

**Use TCP when:**
- Data integrity and ordering are required: web (HTTP), databases, file transfer (FTP, SFTP), email (SMTP), SSH.
- Missing data means corrupted state and retransmission is acceptable latency cost.

**Use UDP when:**
- Low latency is more important than guaranteed delivery: real-time gaming (stale data useless), live video streaming, VoIP.
- The application can tolerate or handle loss better than TCP's retransmit delay.
- DNS (query/response fits in one packet; retrying a new request is faster than TCP retransmit).
- DHCP, SNMP, and other simple request-response protocols where connection overhead is disproportionate.

**Consider QUIC / HTTP/3 when:**
- Building latency-sensitive web applications, especially on mobile or high-latency networks.
- HTTP/2 head-of-line blocking is causing performance problems at the TCP layer.

---

## Common Pitfalls

- **Assuming UDP means "fast TCP"**: UDP is only faster because it removes reliability guarantees. If you add your own reliability layer, you may end up with a TCP-equivalent but poorly implemented.
- **Ignoring TCP head-of-line blocking**: In HTTP/1.1 and HTTP/2 over TCP, a single dropped packet stalls all multiplexed streams. HTTP/3 over QUIC solves this.
- **Forgetting TCP connection setup cost**: For short-lived request/response interactions, the 1+ RTT for SYN/SYN-ACK/ACK + TLS handshake can dominate total latency. Connection pooling and keep-alive are essential.
- **Using UDP without application-level flow control**: UDP can saturate a network link. Without self-imposed rate limiting, a UDP sender can crowd out other traffic and harm itself (increased packet loss, jitter).

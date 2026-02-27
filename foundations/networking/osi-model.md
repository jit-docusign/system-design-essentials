# OSI Model

## What it is

The **OSI (Open Systems Interconnection) Model** is a conceptual framework that standardizes the functions of a communication system into seven abstract layers. It was developed by ISO to allow different systems and vendors to communicate through well-defined interfaces, regardless of their underlying hardware or software.

For system design, the most important layers are **Layer 4 (Transport)** and **Layer 7 (Application)** — this distinction determines what information a load balancer, firewall, or proxy can see and act on.

---

## How it works

### The 7 Layers (Top to Bottom)

```
┌───────────────────────────────────────────────┐
│  7 — Application    (HTTP, DNS, SMTP, FTP)    │  ← User-facing protocols
├───────────────────────────────────────────────┤
│  6 — Presentation   (TLS/SSL, encoding)       │  ← Data format / encryption
├───────────────────────────────────────────────┤
│  5 — Session        (session management)      │  ← Connection lifecycle
├───────────────────────────────────────────────┤
│  4 — Transport      (TCP, UDP)                │  ← End-to-end delivery
├───────────────────────────────────────────────┤
│  3 — Network        (IP, ICMP, routing)       │  ← Logical addressing / routing
├───────────────────────────────────────────────┤
│  2 — Data Link      (Ethernet, MAC)           │  ← Node-to-node delivery
├───────────────────────────────────────────────┤
│  1 — Physical       (cables, radio, fiber)    │  ← Raw bit transmission
└───────────────────────────────────────────────┘
```

| Layer | Name | Protocol Data Unit | Key Protocols | What it does |
|---|---|---|---|---|
| 7 | Application | Message / Data | HTTP, HTTPS, DNS, SMTP, FTP, gRPC | User-facing services and APIs |
| 6 | Presentation | Data | TLS, SSL, JPEG, MPEG | Encryption, encoding, data format translation |
| 5 | Session | Data | NetBIOS, RPC | Managing sessions between applications |
| 4 | Transport | Segment (TCP) / Datagram (UDP) | TCP, UDP, QUIC | Reliable or unreliable end-to-end delivery, ports |
| 3 | Network | Packet | IP (IPv4/IPv6), ICMP | Logical addressing, routing between hosts |
| 2 | Data Link | Frame | Ethernet, Wi-Fi (802.11) | Node-to-node delivery, MAC addresses |
| 1 | Physical | Bits | Copper, fiber, radio | Raw bit transmission |

### The TCP/IP Model (Practical Equivalent)

In practice, the internet uses the **TCP/IP model** (also called the Internet model) which collapses the 7 OSI layers into 4:

```
OSI                   │  TCP/IP
──────────────────────│────────────────────
Layer 7 Application   │  Application
Layer 6 Presentation  │  (HTTP, DNS, TLS)
Layer 5 Session       │
──────────────────────│────────────────────
Layer 4 Transport     │  Transport (TCP, UDP)
──────────────────────│────────────────────
Layer 3 Network       │  Internet (IP)
──────────────────────│────────────────────
Layer 2 Data Link     │  Network Access
Layer 1 Physical      │  (Ethernet, Wi-Fi)
```

### L4 vs L7 — The Critical Distinction for System Design

Almost all system design references to the OSI model reduce to one question: **does the component operate at Layer 4 or Layer 7?**

**Layer 4 (Transport layer)**:
- Sees: source IP, destination IP, source port, destination port, TCP/UDP protocol.
- Cannot see: HTTP headers, path, method, body, cookies.
- Examples: L4 load balancer, NAT, stateful firewall.
- Use: simple TCP/UDP routing based on IP:port. Extremely fast.

**Layer 7 (Application layer)**:
- Sees everything L4 sees, plus: HTTP method, URL path, headers, cookies, body content.
- Can: route based on URL path, modify requests/responses, terminate TLS, inspect content.
- Examples: L7 load balancer (Nginx, HAProxy), API gateway, reverse proxy, WAF.
- Use: content-based routing, A/B testing, authentication enforcement, rate limiting.

```
L4 Load Balancer:
  Client → [LB looks at IP:port] → Backends
  (Fast, but cannot inspect HTTP content)

L7 Load Balancer:
  Client → [LB reads HTTP headers, URL, cookies] → Routes to specific backends
  e.g. /api → API servers    / → Web servers   /static → CDN
  (More capable, slightly more overhead)
```

### Data Encapsulation

As data flows down the layers, each layer adds its own header (and trailer):

```
Application data: "GET /index.html"
  └── Wrapped in HTTP (Layer 7)
      └── Wrapped in TLS (Layer 6)
          └── Wrapped in TCP segment (Layer 4) — adds ports, seq numbers
              └── Wrapped in IP packet (Layer 3) — adds source/dest IP
                  └── Wrapped in Ethernet frame (Layer 2) — adds MAC addresses
                      └── Transmitted as bits (Layer 1)
```

At the destination, each layer strips its header, processing what it understands and passing the rest up.

---

## Key Trade-offs

| Operating Layer | Visibility | Performance | Capability |
|---|---|---|---|
| L3/L4 | IP, port only | Very high | Routing only |
| L7 | Full request context | High (some overhead) | Rich routing, modification, inspection |

---

## When to use

- **L4 routing**: when you need maximum throughput and only need to route by IP/port. Internal service-to-service traffic that doesn't need content inspection.
- **L7 routing**: when you need URL-based routing, auth enforcement, rate limiting, SSL termination, or request modification — virtually all user-facing edge traffic.
- **OSI model as a diagnostic tool**: when debugging network issues, naming the layer helps narrow down the problem. "It's a Layer 3 issue" means routing; "Layer 4" means TCP/UDP; "Layer 7" means the application protocol.

---

## Common Pitfalls

- **Conflating L4 and L7 load balancers**: Choosing an L4 load balancer for a use case requiring URL-path routing is a fundamental architecture mistake.
- **Treating TLS as Layer 7**: TLS typically lives at Layer 6 (Presentation). This matters when deciding where to terminate TLS — at the load balancer (L4 or L7) vs at the application server.
- **Forgetting OSI is a model, not a strict implementation**: The TCP/IP stack doesn't strictly follow OSI. In practice, boundaries blur (e.g., TLS spans L4-L7 depending on framing).

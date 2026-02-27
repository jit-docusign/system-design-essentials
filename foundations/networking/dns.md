# DNS (Domain Name System)

## What it is

The **Domain Name System (DNS)** is the internet's distributed directory service — it translates human-readable domain names (e.g., `www.example.com`) into machine-readable IP addresses (e.g., `93.184.216.34`). Without DNS, users would need to memorize IP addresses for every service they access.

Beyond simple name-to-IP translation, DNS plays a critical role in load distribution, traffic routing, failover, and CDN integration in large-scale systems.

---

## How it works

### Resolution Process

DNS resolution follows a hierarchical query chain:

```
User types: www.example.com

1. Browser checks local DNS cache → not found
2. OS checks /etc/hosts → not found
3. Query sent to Recursive Resolver (usually configured by ISP or set manually, e.g. 8.8.8.8)
4. Recursive Resolver checks its cache → not found
5. Recursive Resolver queries Root Nameserver → returns address of .com TLD nameserver
6. Recursive Resolver queries .com TLD Nameserver → returns address of example.com Authoritative Nameserver
7. Recursive Resolver queries example.com Authoritative Nameserver → returns IP: 93.184.216.34
8. Recursive Resolver caches the result and returns it to the client
9. Client connects to 93.184.216.34
```

### Hierarchy

```
Root Nameservers (.)
    └── TLD Nameservers (.com, .org, .io, ...)
            └── Authoritative Nameservers (example.com)
                    └── Records (A, AAAA, CNAME, MX, TXT, NS, ...)
```

- **Root Nameservers**: 13 logical root nameserver clusters (hundreds of physical servers via anycast). They know where TLD nameservers are.
- **TLD Nameservers**: Managed by registries (Verisign for .com). They know where authoritative nameservers are for each domain.
- **Authoritative Nameservers**: Managed by domain owners. They hold the actual DNS records.
- **Recursive Resolvers**: Client-facing servers that walk the hierarchy on behalf of clients and cache results.

### DNS Record Types

| Record | Purpose | Example |
|---|---|---|
| **A** | Maps domain to IPv4 address | `example.com → 93.184.216.34` |
| **AAAA** | Maps domain to IPv6 address | `example.com → 2606:2800:220:1:...` |
| **CNAME** | Alias from one name to another | `www.example.com → example.com` |
| **MX** | Mail exchange server | `example.com → mail.example.com` |
| **TXT** | Arbitrary text (SPF, DKIM, domain verification) | `"v=spf1 include:..."` |
| **NS** | Authoritative nameserver for the domain | `example.com → ns1.example.com` |
| **PTR** | Reverse DNS (IP → domain name) | Used for email server validation |
| **SRV** | Service discovery (host + port) | Used in SIP, XMPP, Kubernetes |

### TTL (Time To Live)

Every DNS record has a TTL — the time (in seconds) that resolvers and clients should cache the record before re-querying.

```
Low TTL (e.g., 60s): Changes propagate quickly; higher query load on DNS servers
High TTL (e.g., 86400s = 24h): Fewer DNS queries; changes take a long time to propagate
```

**Strategic use of TTL:**
- Set low TTL (60–300s) before a planned migration or failover
- Return to high TTL after stabilization to reduce DNS load
- TTL is the key lever for controlling DNS-based failover speed

### DNS-Based Load Balancing and Traffic Routing

DNS is frequently used at the infrastructure level for traffic distribution:

**Round-robin DNS**: Authoritative nameserver returns multiple IPs for the same domain, cycling through them. Simple but doesn't account for server health or geographic proximity.

**Geo-based routing**: Return different IPs based on the geographic location of the resolver — users in Europe get the European IP, users in Asia get the Asian IP.

**Weighted routing**: Return different IPs with different probability weightings — useful for canary deployments (10% traffic to new version) or gradual traffic migration.

**Failover routing**: Healthcheck the primary endpoint; if it fails, return the IP of the failover endpoint.

**Latency-based routing**: Route clients to the endpoint with the lowest observed latency for their region.

### DNS Caching and Negative Caching

- **Positive caching**: When a record is found, it's cached for TTL seconds.
- **Negative caching**: When a record is **not** found (NXDOMAIN), this is also cached for a period (governed by the SOA record's negative TTL). This prevents repeated queries for non-existent domains.

---

## Key Trade-offs

| Trade-off | Description |
|---|---|
| **High TTL vs low TTL** | High TTL reduces query volume but slows propagation; low TTL enables fast changes but increases DNS server load |
| **DNS LB vs L7 LB** | DNS cannot health-check individual servers; L7 load balancers are superior for fine-grained routing |
| **Caching reduces latency but causes propagation delay** | A record change won't reach all clients until their cached copy expires |

---

## When to use DNS features

- **DNS failover**: suitable for regional-level failover (datacenter level), not server-level (too slow due to TTL).
- **Geo routing**: excellent for directing users to the nearest datacenter.
- **CNAME to CDN**: point your domain to a CDN provider's domain to route traffic through their edge network.
- **DNS-based service discovery**: used in microservice environments (Kubernetes uses internal DNS for service discovery).

---

## Common Pitfalls

- **High TTL before a migration**: If you set TTL to 24h and then need to change the IP, some clients will keep hitting the old IP for up to 24 hours. Pre-warm TTL down to 60–300s before any planned change.
- **Relying on DNS round-robin for load balancing**: DNS doesn't know server health. A failed server keeps getting its IP returned. Use a real load balancer for application-level routing.
- **DNS as a single point of failure**: If your authoritative DNS provider goes down, your entire domain becomes unreachable. Use multiple DNS providers or at minimum multiple NS records.
- **Not caching DNS in applications**: Applications that resolve DNS on every request add unnecessary latency. Use a DNS cache with appropriate TTL respecting behavior.
- **Ignoring negative caching**: A misconfigured record that returns NXDOMAIN will be cached by resolvers — fixing the record doesn't immediately help clients who have cached the negative response.

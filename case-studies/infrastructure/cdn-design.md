# Design a Content Delivery Network (CDN)

## Problem Statement

Design a CDN that accelerates delivery of static and dynamic content to users worldwide by serving it from geographically distributed edge nodes. The CDN must reduce origin server load, reduce latency for end users, and handle traffic spikes. This models how Cloudflare, Akamai, Fastly, and AWS CloudFront work.

**Scale requirements**:
- 100+ Points of Presence (PoPs) globally
- 100 Tbps peak egress bandwidth
- 10 million requests per second
- Edge cache hit rate: > 90%
- p95 latency from edge to user: < 5ms
- Origin offload: serve 90%+ of traffic from cache (reducing origin to 10% of hits)

---

## What It Is

A CDN is a globally distributed network of servers (edge nodes / PoPs) that:
1. Cache content close to users
2. Route requests to the nearest healthy edge
3. Shield the origin server from direct traffic

```
Without CDN:
  User in Tokyo ──────────────────────────── Origin (US East) — 150ms RTT

With CDN:
  User in Tokyo ── Edge PoP Tokyo (5ms) ─── cache hit → response in 5ms
                                        └── cache miss → fetch from Origin (150ms, once)
```

---

## Architecture

### CDN Topology

```
Origin Server (Customer's infrastructure)
        │
   ┌────┴─────────────────────────────────────┐
   │              CDN Network                  │
   │                                          │
   │  Tier 1: Regional Cache (Mid-tier)       │
   │  ┌──────────┐  ┌──────────┐  ┌────────┐ │
   │  │US-East   │  │EU-West   │  │AP-SE   │ │
   │  │Regional  │  │Regional  │  │Regional│ │
   │  └──────────┘  └──────────┘  └────────┘ │
   │       │               │            │    │
   │  Tier 2: Edge PoPs (closest to users)   │
   │  ┌───┐ ┌───┐  ┌───┐ ┌───┐  ┌───┐ ┌───┐│
   │  │NY │ │LA │  │Lon│ │Ams│  │SIN│ │TYO││
   │  └───┘ └───┘  └───┘ └───┘  └───┘ └───┘│
   └────────────────────────────────────────┘
         │               │            │
     User (NY)       User (Lon)   User (TYO)

Cache hierarchy:
  L1: Browser cache (client)
  L2: Edge PoP cache (NVMe SSDs, 50-200 TB per PoP)
  L3: Regional mid-tier cache (larger HDDs, 2-10 PB per region)
  L4: Origin (customer's server)
```

### Anycast Routing

CDNs use **IP Anycast** to route users to the closest PoP:

```
CDN announces the same IP block (e.g., 104.16.0.0/12) from all PoPs simultaneously.
BGP routes traffic to the topologically closest PoP automatically.

User in Tokyo → their ISP's BGP routes to Asia-Pacific PoPs
User in London → routed to European PoPs

No DNS round-robin needed for basic routing (latency-based DNS is a complementary approach).

Alternative: GeoDNS
  - CDN has separate IPs per region
  - DNS resolver returns different IP based on client's IP geo location
  - Coarser than Anycast (city-level vs router-level)
```

---

## How It Works

### 1. Cache Key Design

The cache key determines what counts as the "same" request:

```
Default cache key: scheme + host + path + query string
  https://cdn.example.com/images/logo.png → key = "https://cdn.example.com/images/logo.png"

Custom cache key configurations:
  - Normalize query string order: ?a=1&b=2 == ?b=2&a=1 (avoid duplicate cache entries)
  - Vary by header: Vary: Accept-Encoding → separate cache entries for gzip vs br
  - Vary by cookie: logged-in users get different content (dynamic CDN)
  - Strip tracking params: ?utm_source=twitter is same object as no-param URL
  
Vary header handling:
  GET /index.html HTTP/1.1
  Accept-Encoding: gzip
  → Cache key: "/index.html + gzip variant"
  
  GET /index.html HTTP/1.1  
  Accept-Encoding: br
  → Cache key: "/index.html + brotli variant"  (separate cache entry)
```

### 2. Cache Policies

```python
class CacheDecision:
    """Decide whether and how long to cache a response"""
    
    def should_cache(self, response: dict) -> tuple[bool, int]:
        """Returns (should_cache, ttl_seconds)"""
        
        # Never cache:
        if response["status_code"] in [404, 500, 503]:
            return False, 0
        
        if "Set-Cookie" in response["headers"]:
            return False, 0  # Don't cache personalized responses
        
        if response["method"] != "GET":
            return False, 0  # Only cache GET requests
        
        # Respect Cache-Control from origin
        cc = response["headers"].get("Cache-Control", "")
        
        if "no-store" in cc:
            return False, 0
        
        if "no-cache" in cc:
            return True, 0  # Cache but always revalidate (stale-while-revalidate)
        
        if "max-age" in cc:
            max_age = int(cc.split("max-age=")[1].split(",")[0].strip())
            return True, max_age
        
        # Default: cache for 1 hour if no Cache-Control
        if response["content_type"] in ["image/jpeg", "image/png", 
                                          "application/javascript", "text/css"]:
            return True, 3600
        
        return False, 0
```

### 3. Origin Fetch and Cache Population

```python
import asyncio
import aiohttp

class EdgeNode:
    def __init__(self, cache_store, regional_cache, origin_url):
        self.local_cache = cache_store     # NVMe SSD, ~200 TB
        self.regional_cache = regional_cache  # L3 mid-tier
        self.origin_url = origin_url
    
    async def handle_request(self, request: dict) -> dict:
        cache_key = self.compute_cache_key(request)
        
        # L2: Edge cache
        cached = self.local_cache.get(cache_key)
        if cached and not cached.is_stale():
            return self.serve_from_cache(cached, "HIT")
        
        # L3: Regional mid-tier cache
        regional = await self.regional_cache.get(cache_key)
        if regional and not regional.is_stale():
            self.local_cache.put(cache_key, regional)  # populate L2
            return self.serve_from_cache(regional, "MISS-REGIONAL")
        
        # L4: Origin fetch (cache miss)
        response = await self.fetch_from_origin(request)
        
        should_cache, ttl = CacheDecision().should_cache(response)
        if should_cache and ttl > 0:
            self.local_cache.put(cache_key, response, ttl=ttl)
            await self.regional_cache.put(cache_key, response, ttl=ttl)
        
        return self.add_cache_header(response, "MISS")
    
    async def fetch_from_origin(self, request: dict) -> dict:
        # Request collapsing: if 1,000 concurrent requests for same uncached object,
        # only send ONE request to origin. Others wait for that single response.
        async with self.origin_lock(request["cache_key"]):
            # Check again after acquiring lock (another request may have populated cache)
            cached = self.local_cache.get(request["cache_key"])
            if cached:
                return cached
            
            async with aiohttp.ClientSession() as session:
                async with session.get(self.origin_url + request["path"]) as resp:
                    return {
                        "status_code": resp.status,
                        "headers": dict(resp.headers),
                        "body": await resp.read(),
                        "fetched_at": time.time()
                    }
```

### 4. Cache Invalidation

```python
# Instant purge: invalidate a URL across all PoPs
# This is how Cloudflare/Fastly invalidate cache in <150ms globally

class CachePurgeService:
    def purge_url(self, url: str):
        """
        Fanout purge request to all edge PoPs.
        Each PoP removes the cached entry immediately.
        """
        cache_key = compute_cache_key(url)
        
        # Publish purge event to all PoPs via message bus
        for pop in self.all_pops:
            pop.purge_async(cache_key)  # fire-and-forget to each PoP
        
        # Log the purge for audit trail
        self.purge_log.record({"url": url, "key": cache_key, "timestamp": now()})
    
    def purge_tag(self, cache_tag: str):
        """
        Surrogate keys / cache tags: tag a group of responses at origin time.
        Purge all tagged objects at once.
        
        Origin sets: Surrogate-Key: product-123 category-electronics
        Purge all: purge_tag("product-123") → invalidates all cached responses for that product
        
        Use case: product price changes → purge all pages showing that product
        """
        for pop in self.all_pops:
            pop.purge_tag_async(cache_tag)

# Stale-while-revalidate: serve stale content while fetching fresh version
# Origin sets: Cache-Control: max-age=300, stale-while-revalidate=60
# At t=305s (just expired): serve stale immediately, background fetch new version
# User sees immediate response; next request gets fresh content
```

### 5. TLS Termination at the Edge

```
Without CDN edge TLS:
  User ──── TLS handshake (150ms) ──── Origin in US

With CDN edge TLS:
  User ──── TLS handshake (5ms) ──── Edge PoP nearby
  Edge PoP ──── persistent TLS connection (pre-warmed) ──── Origin
  
TLS 1.3 with 0-RTT session resumption:
  Returning user: 0 additional TLS latency (session ticket reused)
  New user: 1 RTT to nearest edge PoP (vs 1 RTT to distant origin)

CDN manages TLS certificates for customer domains:
  - Certificate issuance via Let's Encrypt or DigiCert
  - Auto-renewal 30 days before expiry
  - OCSP stapling to avoid per-request revocation checks
  - Certificate deployed to all 100 PoPs within 60 seconds
```

### 6. Dynamic Content Acceleration (Non-Cacheable)

CDN isn't only for static content. For dynamic API responses:

```
Optimizations for cache-miss / non-cacheable traffic:

1. Route optimization (SmartRouting):
   - CDN measures latency between PoPs and origin continuously
   - Selects the fastest backbone path (not necessarily shortest geographic)
   - Avoids congested internet paths

2. Connection reuse:
   - Edge PoP maintains persistent HTTP/2 connections to origin
   - Avoids TCP + TLS handshake on every origin request

3. Protocol optimization:
   - Edge ↔ Origin: HTTP/2 multiplexing (multiple requests/connection)
   - Edge ↔ Origin: QUIC over UDP (recovers faster from packet loss than TCP)

4. Compression:
   - Edge transcodes responses: gzip → brotli (br) for better compression
   - Reduces bytes on the user-to-edge leg

5. Header minimization:
   - Strip internal origin headers before sending to user
   - Add security headers (HSTS, X-Frame-Options, CSP)
```

---

## CDN for Video Streaming

```
Video CDN is specialized:
  - Very large objects (5 GB movies)
  - Partial content (HTTP Range requests, HLS segments)
  - Long-tail content (rarely watched videos)

For popular content (top 1%):
  - Pre-push to all PoPs before user requests (predictive)
  - High cache hit rate, low origin load

For long-tail content (99%):
  - Pull-through cache (origin on first request)
  - Store on regional tier (not all PoPs)
  - Object storage (HDDs + prefetch)

HLS segment caching:
  - Each segment: 6 seconds of video, ~1 MB
  - Cache key: segment URL (includes timestamp, so segments change per upload)
  - TTL: very long (segments are immutable once created)
  - Manifest (m3u8): short TTL (30s) — live streams update frequently
```

---

## Key Design Decisions

| Decision | PoP Cache | Regional Cache | Origin |
|---|---|---|---|
| Storage | NVMe SSD (fast, expensive) | HDD (slower, cheap, large) | Customer's infra |
| Capacity per node | 50-200 TB | 2-10 PB per region | Unlimited (customer) |
| Latency | < 1ms (local disk) | ~5ms (in-region network) | 50-200ms (WAN) |
| Cache hit for | Hot content (top 1%) | Warm content (top 20%) | Cold/uncacheable |

---

## Common Interview Mistakes

1. **Not explaining cache key design**: the cache key determines whether two requests hit the same cache entry. Forgetting to discuss Vary headers and query string normalization is a gap.

2. **Ignoring request collapsing**: if 1,000 concurrent users request the same cold object, without collapsing you'd send 1,000 parallel requests to origin. Collapse these into one.

3. **Treating CDN as static-only**: CDNs accelerate dynamic APIs too (via persistent connections, routing optimization). Modern CDNs run code at the edge (Cloudflare Workers, Lambda@Edge).

4. **Not discussing cache invalidation**: "just set a long TTL" is not enough when content changes. Discuss instant purge, stale-while-revalidate, surrogate keys.

5. **Forgetting the multi-tier cache**: edge PoP → regional mid-tier → origin. Without a regional tier, every edge cache miss hits the origin directly.

6. **Overlooking TLS at the edge**: TLS termination at the nearest PoP (not origin) is one of the primary latency benefits of a CDN. Not mentioning this misses a key architectural reason to use a CDN.

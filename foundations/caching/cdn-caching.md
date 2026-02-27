# CDN Caching

## What it is

**CDN (Content Delivery Network) caching** is the practice of storing copies of static and dynamic content at **edge nodes** geographically distributed around the world, so that users receive content from a nearby edge rather than from a distant origin server. CDN caching is the outermost, highest-throughput caching layer in a web architecture.

For a deep dive into CDNs as infrastructure, see [CDN](../traffic/cdn.md). This document focuses specifically on the **caching behaviors** of a CDN and how to configure them correctly.

---

## How it works

### CDN Cache Hit Flow

```
User in Tokyo
    │ GET /images/hero.png
    ▼
CDN Edge Node (Tokyo)
    ├── HIT: image cached → return directly (< 10ms)
    └── MISS: forward to origin → cache response → return (100–500ms)

User in London (same asset)
    │ GET /images/hero.png
    ▼
CDN Edge Node (London)
    ├── May be HIT if another London user fetched first
    └── MISS: forward to origin (or regional PoP) → cache
```

### Cache-Control Headers

The application controls CDN (and browser) caching via HTTP response headers:

```http
# Public — cacheable by CDN and browser for 1 hour
Cache-Control: public, max-age=3600

# Stale-while-revalidate: serve stale for 60s while revalidating in background
Cache-Control: public, max-age=3600, stale-while-revalidate=60

# Private — only browser cache, not CDN
Cache-Control: private, max-age=300

# No caching at all
Cache-Control: no-store

# Validate with origin before serving (not "no-cache" = not "don't cache")
Cache-Control: no-cache
```

### What to Cache at the CDN

| Content type | CDN caching | Rationale |
|---|---|---|
| Static assets (JS, CSS, images, fonts) | ✅ Yes — long TTL (1y with fingerprinting) | Never changes for a given filename |
| API responses (public, immutable) | ✅ Yes — short TTL | Reduce origin load |
| HTML pages (static sites) | ✅ Yes | Same content for all users |
| Personalized API responses (user-specific) | ❌ No | User A's data would be served to User B |
| Authenticated responses | ❌ No | Security risk |
| Real-time data (prices, inventory) | ❌ or very short TTL only | Staleness intolerable |

**The key rule**: only cache responses that are **identical for all users** at a given URL. Responses that vary by user identity, cookie, session, or permissions must never be cached at the CDN without careful `Vary` header management.

### Cache Keys

A CDN determines if a request is a cache hit based on the **cache key** — typically the URL. But URL alone may be insufficient:

```http
GET /api/products?lang=en  → different content than
GET /api/products?lang=fr

GET /api/products  (User-Agent: mobile) → different than
GET /api/products  (User-Agent: desktop)
```

**Vary header** tells the CDN to include specific request headers in the cache key:
```http
Vary: Accept-Language    ← separate cache per language
Vary: Accept-Encoding    ← separate cache per encoding (gzip vs brotli)
Vary: User-Agent         ← dangerous: too many variants, low hit rate
```

Over-using `Vary` fragments the cache, reducing hit rates dramatically. Use `Vary` carefully; prefer URL-based differentiation (`/api/products?lang=en` vs `/api/products?lang=fr`) over header-based.

### Cache Invalidation at CDN

CDN-cached content persists until TTL expires. To force immediate update:

**1. Cache purge / invalidation API**: send an API call to the CDN to remove specific URLs:
```bash
# Cloudflare purge example
curl -X DELETE "https://api.cloudflare.com/client/v4/zones/{zone_id}/purge_cache" \
  -H "Content-Type: application/json" \
  --data '{"files":["https://example.com/api/products"]}'
```

**2. URL fingerprinting (cache busting)**: for static assets, embed a content hash in the filename. When the asset changes, deploy with a new filename — CDN serves the new file without needing to invalidate.
```html
<!-- Build system generates content-hashed filenames -->
<link rel="stylesheet" href="/static/app.7f3b2a.css">
<!-- After change: -->
<link rel="stylesheet" href="/static/app.9c1e4d.css">
```

**3. Versioned URL paths**: use a version prefix in the URL:
```
/v2/api/products  (cache-busts when version changes)
```

**4. Stale-While-Revalidate**: the CDN serves stale content while asynchronously fetching a fresh version — zero-latency cache refresh.

### CDN and Dynamic / Edge Computing

Modern CDNs (Cloudflare Workers, Fastly Compute@Edge, Lambda@Edge) allow running code at the edge:
- Personalization at the edge (vary by geo without full origin request).
- A/B testing without origin latency.
- Authentication at the edge (validate JWT at CDN tier).
- API routing and aggregation.

This blurs the line between CDN and application server — the edge can return dynamic responses without hitting the origin for common cases.

### CDN Cache Hit Rate — Business Impact

```
Typical asset traffic: 90% static (images, JS, CSS), 10% API

Without CDN:
  All 100% requests hit origin servers
  Origin must handle peak traffic (say, 100K req/s)

With CDN (90% hit rate on static):
  90% served from edge
  10% origin requests → 10K req/s on origin (10× reduction)
  Latency improvement: global users see < 20ms instead of 150–500ms
```

---

## Key Trade-offs

| Benefit | Challenge |
|---|---|
| Global low-latency delivery | Stale content without aggressive invalidation |
| Massive traffic absorption (reduces origin load 80–95%) | Debugging cache misses/hits can be complex |
| DDoS mitigation (traffic absorbed at edge) | Personalized content must bypass CDN |
| High availability (edge serves even if origin is down) | CDN costs scale with bandwidth |
| Automatic geographic routing | Vary header misconfiguration causes security/privacy issues |

---

## Common Pitfalls

- **Caching personalized responses**: the single most dangerous CDN mistake. If User A's authenticated response is cached by CDN, User B may receive it. Always set `Cache-Control: private` for user-specific responses.
- **Not fingerprinting static assets**: deploying new JS/CSS to the same filename means CDN may serve the old version for up to TTL duration. All static assets should use content-hash filenames and `max-age: 31536000` (1 year).
- **TTL too short for immutable content**: CSS/JS files with 5-minute TTLs produce constant origin requests for unchanged content. Match TTL to actual content change frequency.
- **Forgetting to purge on content update**: updating a product description but forgetting to purge the CDN means users see old data until TTL expires. Automate CDN purge as part of your deployment or content publish pipeline.
- **Using Vary: User-Agent**: generates one cached version per unique user agent string. Thousands of UA variants → near-zero cache hit rate. Avoid.

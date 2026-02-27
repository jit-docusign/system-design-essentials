# Content Delivery Network (CDN)

## What it is

A **Content Delivery Network (CDN)** is a globally distributed network of **edge servers** (Points of Presence, PoPs) that cache and serve content from locations geographically close to end users. Instead of every user's request traveling to a central origin server, the request is served by the nearest edge node — dramatically reducing latency and offloading traffic from the origin.

CDNs are essential for serving static assets (images, videos, JavaScript, CSS), but modern CDNs also accelerate dynamic content, handle DDoS protection, terminate TLS, and run edge compute.

---

## How it works

### Basic Flow (Pull CDN)

```
First request from user in Singapore:
  User (Singapore) → CDN Edge (Singapore) → MISS → Origin Server (US) → Edge caches content
  Edge → User: response (latency: ~200ms including origin round trip)

Subsequent requests from Singapore for same content:
  User (Singapore) → CDN Edge (Singapore) → HIT → serves from edge cache
  Edge → User: response (latency: ~5ms — no origin involved)
```

The edge acts as a **transparent cache** between users and the origin. Users experience low latency because the CDN edge is geographically close.

### Pull vs Push CDN

#### Pull CDN (Lazy Caching)

Content is **pulled from the origin on first request** and cached at the edge for subsequent requests. The edge is populated on-demand.

```
Flow:
  Client requests /image.jpg
  Edge checks local cache → miss
  Edge fetches /image.jpg from origin
  Edge caches response (obeys Cache-Control headers)
  Edge returns content to client
  All subsequent requests for /image.jpg are served from edge cache until TTL expires
```

**Best for**: Large, varied content libraries where it's impractical to pre-populate edges. Most static asset CDNs work this way.

**Downside**: The first request to each edge (cache miss) pays the full origin round-trip. This "cache warming" latency affects the first user in each region.

#### Push CDN (Proactive Distribution)

Content is **manually or programmatically pushed to CDN edges** before any user requests it.

```
You upload /app.min.js v2.0 to CDN origin
CDN distributes it to all edges immediately
All users worldwide get v2.0 from their local edge on first request
```

**Best for**:
- Predictable content that you know will be needed (software releases, event assets, game patches).
- Content that must be available globally from the first request (no cold-start miss).
- Large files (video, software installers) where a cache miss is very expensive.

**Downside**: Storage is always consumed at all edges, even if a region never requests the content.

### Cache Control and TTL

CDN edges respect HTTP cache headers set by the origin:

| Header | Effect |
|---|---|
| `Cache-Control: max-age=86400` | Cache for 24 hours |
| `Cache-Control: no-store` | Never cache |
| `Cache-Control: private` | Cache only at browser, not CDN |
| `Cache-Control: s-maxage=3600` | CDN-specific TTL (overrides `max-age` for shared caches) |
| `Vary: Accept-Encoding` | Cache separate versions per encoding |
| `Surrogate-Key: product_42` | CDN-proprietary: tag content with keys for selective invalidation |

### Cache Invalidation

When content changes before its TTL expires, it needs to be invalidated (purged) from edges:

- **TTL expiry**: the simplest approach — set a short TTL and let it expire naturally. Safe but may serve stale content briefly.
- **URL versioning / cache busting**: change the URL when content changes (`/app.v2.0.min.js`), bypassing the old cached version entirely. The old URL's cache expires naturally.
- **Explicit purge API**: CDN providers offer APIs to purge specific URLs or sets of URLs immediately.
- **Surrogate-key / tag-based invalidation**: Tag content with logical keys (e.g., `product_42`); when the product changes, purge all content tagged `product_42` with a single API call — even if it's spread across thousands of URLs.

Invalidation propagates to all edge PoPs — but propagation is not instantaneous. During the propagation window, some edges may still serve stale content.

### Origin Shielding

Without shielding, when a popular piece of content expires simultaneously across 100 edge PoPs, all 100 edges may simultaneously send a request to the origin — a **thundering herd**. **Origin shielding** designates one intermediate PoP as the primary cache layer between edges and the origin:

```
Without shielding:           │  With shielding:
Edge PoP (US) → Origin       │  Edge PoP (US) → Shield PoP → Origin
Edge PoP (EU) → Origin       │  Edge PoP (EU) → Shield PoP (cache hit)
Edge PoP (AP) → Origin       │  Edge PoP (AP) → Shield PoP (cache hit)
(3 origin requests per miss) │  (1 origin request per miss)
```

### Dynamic Content Acceleration

Beyond static caching, modern CDNs improve performance for uncacheable dynamic content:
- **Persistent connections** between CDN PoPs and origin reduce TCP/TLS overhead.
- **Route optimization**: CDN PoPs communicate over private, optimized backbone networks rather than the public internet.
- **Protocol acceleration**: HTTP/2 and HTTP/3 termination at the edge, even if the origin only supports HTTP/1.1.

### Edge Compute

Modern CDNs support running code at the edge (e.g., Cloudflare Workers, Fastly Compute):
- Personalize responses at the edge (A/B testing, locale-specific rendering).
- Enforce authentication/authorization close to the user.
- Manipulate requests/responses without round-tripping to origin.

---

## Key Trade-offs

| Trade-off | Description |
|---|---|
| **Cache hit rate vs freshness** | Longer TTL = better hit rates, lower origin load; shorter TTL = fresher content, more origin requests |
| **Push vs Pull** | Push gives deterministic cold-start performance; Pull is simpler to operate for large content libraries |
| **Global distribution vs cost** | More PoPs = lower latency globally; costs more in CDN fees |
| **Cache invalidation complexity** | Aggressive caching requires robust invalidation; getting invalidation wrong serves wrong/stale content |

---

## When to use

- **Static assets** (images, CSS, JS, fonts): always use a CDN — there is no good reason to serve these from an origin server.
- **Video / large files**: CDN is essential for streaming at scale; origin bandwidth alone is prohibitively expensive.
- **Global products**: without a CDN, users in Asia or Europe suffer 200–400ms extra latency hitting an origin in the US. A CDN makes your site feel local everywhere.
- **DDoS protection**: CDN PoPs absorb and filter volumetric attacks before they reach origin.

---

## Common Pitfalls

- **Caching private/authenticated content**: content with `Cache-Control: private` or personalized responses must never be cached at the CDN edge (they'd be served to the wrong user). Always validate cache headers carefully.
- **No cache busting**: static assets served without versioned URLs get stuck in CDN caches for their full TTL even after deployment. Use content-hashed filenames (`/app.8f3a1b.min.js`).
- **Forgetting to configure cache headers on the origin**: if the origin sends no `Cache-Control` header, CDN behavior is undefined — some CDNs cache with a default TTL, others don't. Be explicit.
- **Assuming invalidation is instant**: CDN purge propagation takes from seconds to minutes. Design systems that can tolerate a brief window of stale content after invalidation.
- **Bypassing the CDN for API traffic**: CDNs can also cache API responses that are public and stable (e.g., product catalogs). Not using CDN for cacheable API responses wastes a significant latency and load improvement opportunity.

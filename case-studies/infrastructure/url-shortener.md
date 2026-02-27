# Design a URL Shortener (bit.ly / TinyURL)

## Problem Statement

Design a URL shortening service that takes a long URL (e.g., `https://www.example.com/very/long/path?with=query&params=here`) and returns a short alias (e.g., `https://bit.ly/abc123`). When users visit the short URL, they are instantly redirected to the original URL.

**Scale requirements**:
- 100 million URLs shortened per day (write: ~1,200 URLs/sec)
- 10 billion redirects per day (read: ~115,000 redirects/sec)
- URLs never expire (optional: configurable TTL)
- Short codes: 7 characters of Base62 → 62^7 ≈ 3.5 trillion unique codes
- Redirect latency: < 10ms p99

---

## What It Is

A URL shortener has two operations:
1. **Shorten**: `POST /shorten {url: "..."}` → returns short code
2. **Redirect**: `GET /{code}` → `HTTP 301/302 Location: <original_url>`

While the concept is simple, the engineering challenges that make it non-trivial are:
- Collision-free code generation at scale
- Read-heavy performance (100:1 read/write ratio or higher)
- Analytics tracking

---

## Short Code Generation

### Option A: Hash-Based

```python
import hashlib
import base64

def hash_url(long_url: str) -> str:
    """
    Compute MD5 of URL → take first 7 Base62 chars.
    """
    md5 = hashlib.md5(long_url.encode()).hexdigest()
    # Convert hex to integer, encode in Base62
    n = int(md5[:16], 16)  # first 64 bits of MD5
    return to_base62(n)[:7]

def to_base62(n: int) -> str:
    CHARS = "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz"
    result = []
    while n > 0:
        result.append(CHARS[n % 62])
        n //= 62
    return ''.join(reversed(result)).zfill(7)

# Problem: Hash collisions
# hash_url("https://example.com") = "aB3cDe7"
# hash_url("https://different.com") = "aB3cDe7"  ← collision!
# Two different URLs map to the same code.

# Collision resolution: append counter suffix
def hash_url_no_collision(long_url: str, suffix: int = 0) -> str:
    candidate = hash_url(long_url + str(suffix))[:7]
    if not db.exists(candidate):
        return candidate
    return hash_url_no_collision(long_url, suffix + 1)  # retry with suffix
```

**Downsides of hashing**:
- Same long URL always generates same short code (good for dedup, but requires check)
- Collision detection requires a DB lookup before every insertion
- Under high concurrency, two requests for the same URL can race

### Option B: Counter-Based (Distributed Counter)

```python
# Global auto-increment counter → convert to Base62

class URLShortener:
    def __init__(self, db, redis_client):
        self.db = db
        self.redis = redis_client
    
    def shorten(self, long_url: str, user_id: str = None) -> str:
        # 1. Check if URL was already shortened (dedup)
        existing = self.db.query(
            "SELECT short_code FROM urls WHERE long_url_hash = %s",
            [hash64(long_url)]
        )
        if existing:
            return existing[0]["short_code"]
        
        # 2. Get next ID from distributed counter
        url_id = self.redis.incr("url_counter")
        
        # 3. Convert ID to Base62 short code
        short_code = to_base62(url_id)
        
        # 4. Store in DB
        self.db.execute("""
            INSERT INTO urls (id, short_code, long_url, long_url_hash, user_id, created_at)
            VALUES (%s, %s, %s, %s, %s, NOW())
            ON CONFLICT (long_url_hash) DO NOTHING
        """, [url_id, short_code, long_url, hash64(long_url), user_id])
        
        # 5. Cache for fast redirect
        self.redis.setex(f"url:{short_code}", 86400, long_url)  # 24h cache
        
        return short_code

# ID → Base62 mapping:
# ID=1       → "0000001"
# ID=62      → "0000010"
# ID=3549145 → "abc1234"
# ID=3.5T    → "zzzzzzz" (maximum 7-char code)

def to_base62(n: int, length: int = 7) -> str:
    BASE = "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz"
    result = []
    while n > 0:
        result.append(BASE[n % 62])
        n //= 62
    code = ''.join(reversed(result))
    return code.zfill(length)  # left-pad to 7 chars
```

**Counter-based approach is preferred** — no collisions, sequential IDs are predictable (which  can be a security concern though: `abc123` → `abc124` might be next).

### Option C: Pre-Generated Code Pool

```
Pre-generate millions of unique random codes offline → store in a "code pool" table.
At shorten time: pick one from pool, mark as used.

Advantages:
- No collision detection at runtime
- Easy to avoid sequential/predictable codes

Code pool table:
  code_pool:
    code VARCHAR(7) PRIMARY KEY
    status ENUM('available', 'used')
    
  At shorten time:
    SELECT code FROM code_pool WHERE status = 'available' LIMIT 1 FOR UPDATE
    UPDATE code_pool SET status = 'used' WHERE code = ?
    INSERT INTO urls ...

Disadvantage: Pre-generation job required; "FOR UPDATE" still serializes at DB.
```

---

## Redirect Service (Hot Path)

The redirect path must be extremely fast — 115,000 requests/sec:

```python
from flask import Flask, redirect, abort
import redis

app = Flask(__name__)
r = redis.Redis()

@app.route("/<short_code>")
def redirect_to_long(short_code: str):
    # Validate short code format first (security)
    if not short_code.isalnum() or len(short_code) != 7:
        abort(400)
    
    # 1. Check Redis cache (L1)
    long_url = r.get(f"url:{short_code}")
    
    if not long_url:
        # 2. Cache miss → check DB
        row = db.query("SELECT long_url, is_active FROM urls WHERE short_code = %s", 
                       [short_code])
        if not row or not row[0]["is_active"]:
            abort(404)
        
        long_url = row[0]["long_url"]
        
        # Re-populate cache
        r.setex(f"url:{short_code}", 86400, long_url)
    else:
        long_url = long_url.decode()
    
    # 3. Async analytics (don't block redirect)
    analytics_queue.push_async({
        "short_code": short_code,
        "timestamp": time.time(),
        "ip": request.remote_addr,
        "referrer": request.referrer,
        "user_agent": request.user_agent.string
    })
    
    # 4. Redirect
    return redirect(long_url, code=302)  # 302 = temporary (browser won't cache)
    # vs 301 = permanent (browser caches forever — can't track clicks, can't update)
```

### 301 vs. 302 Redirect

| | 301 Permanent | 302 Temporary |
|---|---|---|
| Browser caches? | Yes (forever) | No |
| Click tracking | No (browser skips your server) | Yes (every click hits your server) |
| SEO impact | Link equity transfers to destination | No transfer |
| Use when | SEO optimization only | Analytics-required (most shorteners) |

---

## Data Model

```sql
CREATE TABLE urls (
    id              BIGSERIAL PRIMARY KEY,
    short_code      VARCHAR(10) UNIQUE NOT NULL,
    long_url        TEXT NOT NULL,
    long_url_hash   BIGINT NOT NULL,            -- for dedup lookup
    user_id         BIGINT,                      -- NULL for anonymous
    title           TEXT,                        -- scraped page title
    is_active       BOOLEAN DEFAULT TRUE,
    expires_at      TIMESTAMPTZ,                 -- NULL = never expires
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_long_url_hash ON urls(long_url_hash);   -- dedup lookups
CREATE INDEX idx_user_id ON urls(user_id);               -- user's link list
CREATE INDEX idx_expires_at ON urls(expires_at) WHERE expires_at IS NOT NULL;

-- Click analytics (write stream, not queryable inline)
CREATE TABLE clicks (
    id          BIGSERIAL PRIMARY KEY,
    short_code  VARCHAR(10) NOT NULL,
    clicked_at  TIMESTAMPTZ NOT NULL,
    country     VARCHAR(2),     -- GeoIP lookup
    device_type VARCHAR(20),    -- mobile/desktop/tablet
    referrer    TEXT,
    ip_hash     VARCHAR(64)     -- hashed for privacy (not raw IP)
);
-- In practice: clicks table is write-only → pipe to Kafka → OLAP for analytics
```

---

## Distributed Counter Approach

Single Redis INCR is a bottleneck at high scale. Use ID ranges:

```python
class RangeBasedCounter:
    """
    Each application server claims a range of 1,000 IDs from a central counter.
    Then uses them locally without hitting Redis for each request.
    """
    
    def __init__(self, server_id: str, range_size: int = 1000):
        self.server_id = server_id
        self.range_size = range_size
        self.current_id = None
        self.range_end = None
    
    def next_id(self) -> int:
        # If we've exhausted our range, claim a new one
        if self.current_id is None or self.current_id >= self.range_end:
            self._claim_range()
        
        id_val = self.current_id
        self.current_id += 1
        return id_val
    
    def _claim_range(self):
        # Atomically increment global counter by range_size
        end = redis.incrby("url_counter", self.range_size)
        self.range_end = end
        self.current_id = end - self.range_size
        # Server now has IDs [current_id, range_end) locally — no Redis needed per URL

# Alternative: Use Snowflake-like IDs (timestamp + server_id + sequence)
# Guarantees unique IDs across servers without coordination
import time

class SnowflakeID:
    EPOCH = 1704067200  # 2024-01-01 00:00:00 UTC
    
    def __init__(self, server_id: int):  # server_id: 0-1023
        self.server_id = server_id & 0x3FF  # 10 bits
        self.sequence = 0
        self.last_ms = -1
    
    def next_id(self) -> int:
        ms = int((time.time() - self.EPOCH) * 1000)
        if ms == self.last_ms:
            self.sequence = (self.sequence + 1) & 0xFFF  # 12 bits
            if self.sequence == 0:
                while ms <= self.last_ms:  # wait for next millisecond
                    ms = int((time.time() - self.EPOCH) * 1000)
        else:
            self.sequence = 0
        self.last_ms = ms
        return (ms << 22) | (self.server_id << 12) | self.sequence
```

---

## Analytics Pipeline

```
Every click → Kafka topic "url.clicks"
    │
    ├── Flink real-time aggregation:
    │     - Clicks per short_code in last 5 minutes → Redis ZSet "trending_links"
    │     - Writes to: clicks_realtime table (ClickHouse)
    │
    └── Batch (daily, Spark):
          - Clicks by country, device, referrer, time-of-day
          - Stored in: OLAP (ClickHouse or BigQuery)
          - Surfaced via: dashboard API for link owners

User sees analytics: how many clicks, from where, on which devices.
```

---

## Key Design Decisions

| Decision | Counter-based | Hash-based | Use |
|---|---|---|---|
| Collision risk | None (sequential) | Possible (need retry) | Counter |
| Predictability | Sequential (security risk) | Random appearance | Hash or UUID |
| Deduplication | Requires separate check | Natural (same URL = same hash) | Hash if dedup needed |
| DB writes | One per shorten | One + collision retry | Counter (simpler) |

---

## Hard-Learned Engineering Lessons

1. **Not discussing the 301 vs. 302 redirect tradeoff**: 301 is permanent and browsers cache it (you lose click tracking). 302 is always correct for analytics.

2. **Forgetting the Redis cache**: serving redirects from the DB alone cannot handle 115K req/sec. Redis cache in front of the DB is essential.

3. **Using MD5 without collision handling**: MD5 collisions are unlikely but possible at billions of URLs. Must have retry or fallback.

4. **Not addressing deduplication**: should `bit.ly/abc` and `bit.ly/xyz` both point to the same long URL if the same URL is shortened twice? Most shorteners return the same code; this requires a `long_url_hash` lookup.

5. **Not considering URL validation/security**: short URLs can point to malware. Discuss URL blocklist checking (Google Safe Browsing API) and not following redirects when shortening.

6. **Using sequential codes without any obfuscation**: purely sequential Base62 codes (`0000001`, `0000002`) allow enumeration of all links. Use a shuffling function or add randomness.

7. **Forgetting expiration/cleanup**: URLs that expire need to be cleaned up (soft delete + background job), and the cache entry must be invalidated.

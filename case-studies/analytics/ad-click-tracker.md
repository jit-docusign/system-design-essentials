# Design an Ad Click Tracking System

## Problem Statement

Design the backend system that records when users click on digital advertisements. This data determines how much advertisers pay, which ads are performing, and feeds into fraud detection. Used by Google Ads, Facebook Ads, Twitter Ads.

**Scale requirements**:
- 10 billion ad impressions per day
- 1 billion ad clicks per day (~11,600 clicks/sec)
- Exactly-once counting: each click counted exactly once (billing accuracy)
- Real-time: click data available within 5 seconds for fraud scoring
- Reporting: advertisers see click counts updated within 1 minute
- 99.999% durability: no click data can be lost (financial data)

---

## What Makes Ad Click Tracking Hard

1. **Exactly-once semantics**: a double-counted click = advertiser overcharged. An uncounted click = publisher underpaid.
2. **High write throughput**: 11,600 clicks/sec with burst peaks 10× higher.
3. **Click fraud prevention**: ~15% of all clicks are fraudulent (bots). Must detect and filter.
4. **Fast landing page redirect**: user clicks → redirect to advertiser's site in < 100ms.

---

## Architecture

```
User Browser                                           
    │                                                  
    │  User clicks ad → GET /click?ad_id=ABC&publisher=XYZ&token=...
    │                                                  
    ▼                                                  
┌─────────────────────────────────────────────────────┐
│  Click Handler (CDN Edge or API Gateway)             │
│  - Validate click token (prevent CSRF/bot clicks)   │
│  - Fire-and-forget to Kafka                          │
│  - Immediately redirect user to advertiser site      │
└───────────────────────────┬─────────────────────────┘
                            │ async write (Kafka)
                            ▼
                ┌────────────────────────┐
                │  Kafka                 │
                │  Topic: ad.clicks.raw  │
                │  (immutable log)       │
                └─────────────┬──────────┘
                              │
            ┌─────────────────┼──────────────────────┐
            │                 │                      │
            ▼                 ▼                      ▼
    ┌──────────────┐  ┌──────────────┐      ┌────────────────┐
    │  Flink       │  │  Flink       │      │  Raw Event     │
    │  Dedup +     │  │  Fraud       │      │  Storage       │
    │  Aggregator  │  │  Detector    │      │  (S3 Parquet)  │
    └──────┬───────┘  └──────┬───────┘      └────────────────┘
           │                 │
           ▼                 ▼
    ┌──────────────┐  ┌──────────────┐
    │  ClickHouse  │  │  Fraud Score │
    │  (OLAP)      │  │  DB          │
    │  Click counts│  │  (blocked    │
    │  per ad/time │  │   IPs, users)│
    └──────────────┘  └──────────────┘
           │
           ▼
    ┌──────────────────────────────────┐
    │  Billing Service (Hourly Batch)  │
    │  Reads VALID click counts        │
    │  → Charge advertiser accounts    │
    └──────────────────────────────────┘
```

---

## How It Works

### 1. Click Token — Anti-Fraud

Clicks are pre-authorized with a signed token generated when the ad is served:

```python
import hmac
import hashlib
import time

SECRET_KEY = "your-secret-key-here"

def generate_click_token(ad_id: str, user_id: str, placement_id: str) -> str:
    """
    Sign a click token when the ad is SERVED (impression time).
    Valid for 10 minutes (prevents replay of old clicks).
    """
    expires_at = int(time.time()) + 600  # 10 minutes
    
    payload = f"{ad_id}|{user_id}|{placement_id}|{expires_at}"
    
    signature = hmac.new(
        SECRET_KEY.encode(),
        payload.encode(),
        hashlib.sha256
    ).hexdigest()[:16]
    
    return f"{payload}|{signature}"

def validate_click_token(token: str) -> dict | None:
    """
    Validate click token on click handler.
    Returns None if invalid/expired.
    """
    try:
        parts = token.split("|")
        ad_id, user_id, placement_id, expires_at, signature = parts
        
        # Check expiry
        if int(expires_at) < time.time():
            return None  # Token expired
        
        # Verify signature
        payload = f"{ad_id}|{user_id}|{placement_id}|{expires_at}"
        expected_sig = hmac.new(
            SECRET_KEY.encode(),
            payload.encode(),
            hashlib.sha256
        ).hexdigest()[:16]
        
        if not hmac.compare_digest(signature, expected_sig):
            return None  # Tampered token
        
        return {"ad_id": ad_id, "user_id": user_id, "placement_id": placement_id}
    
    except Exception:
        return None
```

### 2. Click Handler — Redirect ASAP

```python
from fastapi import FastAPI, Response
from fastapi.responses import RedirectResponse
import asyncio

app = FastAPI()

@app.get("/click")
async def handle_click(
    ad_id: str,
    token: str,
    request: Request
):
    # 1. Validate token BEFORE doing anything (rejects bots with no token)
    token_data = validate_click_token(token)
    if not token_data:
        # Still redirect (don't reveal fraud detection logic), just don't count
        destination = get_ad_destination(ad_id)
        return RedirectResponse(url=destination, status_code=302)
    
    # 2. Fire click event asynchronously (don't block the redirect)
    click_event = {
        "click_id": str(uuid.uuid4()),
        "ad_id": ad_id,
        "user_id": token_data["user_id"],
        "placement_id": token_data["placement_id"],
        "ip": request.client.host,
        "user_agent": request.headers.get("user-agent"),
        "referrer": request.headers.get("referer"),
        "timestamp_ms": int(time.time() * 1000),
        "token": token  # Include token for dedup check later
    }
    
    # Publish to Kafka async (non-blocking)
    asyncio.create_task(kafka_producer.send("ad.clicks.raw", click_event))
    
    # 3. Redirect immediately (user experience first)
    destination = get_ad_destination(ad_id)
    return RedirectResponse(url=destination, status_code=302)
    
    # Note: RedirectResponse returns to browser before Kafka write completes
    # This is acceptable — durability is handled by Kafka replication, not HTTP response
```

### 3. Exactly-Once Deduplication

Clicks can be duplicated by:
- Network retry: user's browser retries the click HTTP request
- Bot scripts: same click URL called multiple times
- Client SDK retries

```python
# Flink deduplication job using event-time windowing + state

from pyflink.datastream import StreamExecutionEnvironment
from pyflink.datastream.state import ValueStateDescriptor

class ClickDeduplicator(KeyedProcessFunction):
    """
    For each (ad_id, user_id) pair: only count one click per 5-minute window.
    Two clicks on the same ad by the same user within 5 minutes → count as one.
    """
    
    def open(self, ctx):
        # State: last click time per (ad_id, user_id)
        self.last_click_time = self.get_runtime_context().get_state(
            ValueStateDescriptor("last_click_time", Types.LONG())
        )
    
    def process_element(self, click: dict, ctx: Context, out: Collector):
        last_time = self.last_click_time.value()
        current_time = click["timestamp_ms"]
        
        DEDUP_WINDOW_MS = 5 * 60 * 1000  # 5 minutes
        
        if last_time is None or (current_time - last_time) > DEDUP_WINDOW_MS:
            # This click is NOT a duplicate — emit it
            self.last_click_time.update(current_time)
            out.collect(click)
        else:
            # Duplicate click within 5-minute window — discard
            metrics.increment("click.deduplicated")

# Also deduplicate by click_id (exact duplicate):
# Use a Bloom filter over recent click_ids (last 1 hour)
from bloomfilter import BloomFilter

click_id_bloom = BloomFilter(max_elements=50_000_000, error_rate=0.001)

def is_exact_duplicate(click_id: str) -> bool:
    if click_id_bloom.add(click_id):
        return False  # New click_id, not a duplicate
    return True       # Already seen this exact click_id
```

### 4. Fraud Detection

```python
class FraudDetector(ProcessFunction):
    FRAUD_SIGNALS = [
        "click_rate_per_ip",      # > 100 clicks/hour from same IP
        "click_rate_per_user",    # > 50 clicks/hour from same user
        "impossible_geography",   # user clicks from two countries within 5 minutes
        "no_valid_token",         # click without a proper sign ad-serve token
        "datacenter_ip",          # click from cloud provider IP range (AWS, GCP, etc.)
        "headless_browser",       # user-agent indicates Puppeteer/Playwright/PhantomJS
        "click_through_time",     # time from ad served to click < 100ms (inhuman speed)
    ]
    
    def process_element(self, click: dict, ctx: Context, out: Collector):
        fraud_score = 0.0
        
        # Check IP click rate (last 1 hour)
        ip_clicks = redis.incr(f"fraud:ip:{click['ip']}:clicks")
        redis.expire(f"fraud:ip:{click['ip']}:clicks", 3600)
        if ip_clicks > 100:
            fraud_score += 0.6
        
        # Check user click rate
        user_clicks = redis.incr(f"fraud:user:{click['user_id']}:clicks")
        redis.expire(f"fraud:user:{click['user_id']}:clicks", 3600)
        if user_clicks > 50:
            fraud_score += 0.5
        
        # Check datacenter IP (pre-loaded IP range list)
        if is_datacenter_ip(click["ip"]):
            fraud_score += 0.8
        
        # Check headless browser signals
        ua = click.get("user_agent", "")
        if any(bot in ua.lower() for bot in ["headless", "phantomjs", "python-requests", "curl"]):
            fraud_score += 0.9
        
        # Emit to appropriate output
        if fraud_score >= 0.7:
            out.collect_to_side("invalid_clicks", {**click, "fraud_score": fraud_score})
        else:
            out.collect({**click, "fraud_score": fraud_score})  # valid clicks
```

### 5. Click Storage and Aggregation

```sql
-- ClickHouse: click events table
CREATE TABLE ad_clicks (
    click_id        UUID,
    ad_id           String,
    advertiser_id   String,
    campaign_id     String,
    placement_id    String,
    user_id         String,
    timestamp       DateTime64(3),
    country         LowCardinality(String),
    device_type     LowCardinality(String),
    is_valid        Bool DEFAULT TRUE,
    fraud_score     Float32
) ENGINE = MergeTree()
  PARTITION BY toYYYYMMDD(timestamp)
  ORDER BY (advertiser_id, timestamp)
  TTL timestamp + INTERVAL 90 DAY;

-- Materialized view: clicks per ad per minute (for dashboards)
CREATE MATERIALIZED VIEW ad_clicks_per_minute
ENGINE = SummingMergeTree()
ORDER BY (ad_id, minute)
AS SELECT
    ad_id,
    advertiser_id,
    toStartOfMinute(timestamp) AS minute,
    countIf(is_valid) AS valid_clicks,
    countIf(NOT is_valid) AS invalid_clicks,
    uniq(user_id) AS unique_users
FROM ad_clicks
GROUP BY ad_id, advertiser_id, minute;

-- Billing query: valid clicks by advertiser this hour
SELECT advertiser_id, sum(valid_clicks) AS billable_clicks
FROM ad_clicks_per_minute
WHERE minute >= now() - INTERVAL 1 HOUR
GROUP BY advertiser_id;
```

---

## Billing Accuracy

```
Billing must use batch-computed counts (not real-time estimates):

1. Real-time dashboard: "~12,345 clicks (live estimate)" 
   → From ClickHouse materialized views (may include late/duplicate events not yet filtered)

2. Billing (hourly batch job):
   → Reads from S3 raw events (at-least-once from Kafka)
   → Runs deduplication by click_id (exact match)
   → Runs fraud filtering (final model, not streaming approximation)
   → Produces final billable count
   → Writes to billing DB: {advertiser_id, hour, billable_clicks, amount_usd}

This two-pass approach ensures billing is always from authoritative, deduplicated data.
Advertisers are billed on the batch count, not the real-time estimate.
```

---

## Key Design Decisions

| Decision | Choice | Reasoning |
|---|---|---|
| Redirect latency | Fire-and-forget Kafka write | Users can't wait for Kafka ACK; redirect first, persist async |
| Deduplication strategy | Bloom filter (5-min window) | Exact dedup impossible at scale; 5-min window catches most retries/bots |
| Fraud detection | Streaming (near-real-time) + batch reprocessing | Streaming for fast blocking; batch for accuracy (catches sophisticated patterns) |
| Billing source of truth | Batch reprocessing from S3 | Real-time can miss late events; batch is exact |
| Click token signing | HMAC-SHA256 (short expiry) | Prevent click injection without impression; expiry prevents replay |

---

## Common Interview Mistakes

1. **Synchronous Kafka write before redirect**: making the user wait for Kafka to confirm will add 10-50ms to redirect latency. Fire-and-forget then redirect.

2. **Not addressing deduplication**: network retries are inevitable. Without dedup, advertisers are double-charged. Bloom filter + time-window dedup is the standard approach.

3. **Using a transactional DB for click writes**: a relational DB at 11,600 writes/sec with ACID guarantees will be the bottleneck. Kafka → OLAP is the right architecture.

4. **Conflating real-time estimates with billing figures**: dashboards show approximate real-time data; billing uses exact batch-processed counts. Mixing these causes disputes with advertisers.

5. **Not discussing click fraud**: ~15% of clicks are fraudulent. An ad click tracker without fraud detection is not production-ready. Describe at least 3-4 fraud signals.

6. **Forgetting late event handling**: mobile apps go offline and fire click events hours later. Event-time processing (not ingestion-time) ensures clicks are attributed to the correct window.

7. **Not addressing idempotency of billing**: the billing batch job might run twice (retry after failure). Without idempotency keys, advertisers get double-billed. Upsert by (advertiser_id, hour) prevents this.

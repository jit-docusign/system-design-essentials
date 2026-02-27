# Design a Flash Sale System

## Problem Statement

Design a system to sell 10,000 limited-edition products in a time-limited sale event (e.g., a sneaker drop, Black Friday deal, or concert tickets). The moment the sale opens, 1 million users simultaneously try to purchase. The system must:
- Allow only 10,000 purchases (no oversell)
- Handle 1 million concurrent requests gracefully
- Respond within 200ms (even to users who don't get the item)
- Prevent bots and unfair advantages

---

## The Core Problem

A flash sale is the hardest case of inventory management:
- Normal e-commerce: a few concurrent purchases per item
- Flash sale: 100× overshoot (1M users for 10K items)

The naive approach — decrement a DB counter on each request — deadlocks under this load.

```
Naive (BROKEN):
  1M concurrent requests → DB UPDATE SET stock = stock - 1 WHERE stock > 0
  → Database row lock contention → p99 latency = 30+ seconds → timeouts → lost sales
```

---

## Architecture

```
1M Users
   │
   │ (pre-sale: static product page, no DB hits)
   ▼
┌────────────────────────────────────────────────────────────────────┐
│  CDN Edge Cache                                                    │
│  Static product page, countdown timer, images                     │
│  Cache-Control: max-age=300 (5 min)                                │
└───────────────────────────┬────────────────────────────────────────┘
                            │ (countdown ends → sale starts)
                            ▼
                  ┌─────────────────────┐
                  │   API Gateway        │
                  │  + Rate Limiter      │◄─── Bot protection
                  │  + Auth (JWT)        │     (token bucket per user)
                  └──────────┬──────────┘
                             │
                  ┌──────────▼──────────┐
                  │  Request Queue       │◄─── Token Queue
                  │  (Kafka / SQS)       │     Emit purchase tokens
                  │                      │     to first 10,000 users
                  └──────────┬──────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
              ▼              ▼              ▼
         ┌─────────┐   ┌─────────┐   ┌─────────┐
         │ Worker  │   │ Worker  │   │ Worker  │
         │ Pool    │   │ Pool    │   │ Pool    │
         └────┬────┘   └────┬────┘   └────┬────┘
              │              │              │
              └──────────────┼──────────────┘
                             │
                    ┌────────▼────────┐
                    │  Redis          │
                    │  stock counter  │
                    │  sold_set       │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │   Orders DB     │
                    │  (PostgreSQL)   │
                    └─────────────────┘
```

---

## How It Works

### Phase 1: Pre-Sale (Before T=0)

```python
# Warm up Redis stock counter before sale opens
def initialize_sale(sale_id: str, stock: int, opens_at: datetime):
    pipe = redis.pipeline()
    
    # Stock counter: start at 10,000
    pipe.set(f"sale:{sale_id}:stock", stock)
    
    # Sold set: track who already purchased (prevent duplicates)
    pipe.delete(f"sale:{sale_id}:sold")  # empty set
    
    # Sale metadata
    pipe.hmset(f"sale:{sale_id}:meta", {
        "opens_at": opens_at.timestamp(),
        "stock": stock,
        "status": "pending"
    })
    
    pipe.execute()

# Pre-load product page to CDN so 1M users see static page cheaply
publish_to_cdn("sale-product-page", cached_html, ttl=300)
```

### Phase 2: Atomic Inventory Reservation in Redis

The key to preventing oversell: Lua script for atomic check-and-decrement:

```lua
-- Redis Lua script (runs atomically, no race conditions)
-- KEYS[1] = stock key (e.g., "sale:abc:stock")
-- KEYS[2] = sold set key (e.g., "sale:abc:sold")
-- ARGV[1] = user_id
-- Returns: 1=success, 0=sold_out, -1=already_purchased

local user_id = ARGV[1]
local stock_key = KEYS[1]
local sold_key = KEYS[2]

-- Anti-duplicate: check if user already purchased
local already_bought = redis.call('SISMEMBER', sold_key, user_id)
if already_bought == 1 then
    return -1  -- already purchased
end

-- Check and decrement stock
local current_stock = tonumber(redis.call('GET', stock_key))
if current_stock == nil or current_stock <= 0 then
    return 0  -- sold out
end

redis.call('DECRBY', stock_key, 1)
redis.call('SADD', sold_key, user_id)

return 1  -- reservation successful
```

```python
RESERVE_SCRIPT = """
local user_id = ARGV[1]
local stock_key = KEYS[1]
local sold_key = KEYS[2]
local already_bought = redis.call('SISMEMBER', sold_key, user_id)
if already_bought == 1 then return -1 end
local stock = tonumber(redis.call('GET', stock_key))
if stock == nil or stock <= 0 then return 0 end
redis.call('DECRBY', stock_key, 1)
redis.call('SADD', sold_key, user_id)
return 1
"""

reserve_lua = redis.register_script(RESERVE_SCRIPT)

def reserve_item(sale_id: str, user_id: str) -> str:
    result = reserve_lua(
        keys=[f"sale:{sale_id}:stock", f"sale:{sale_id}:sold"],
        args=[user_id]
    )
    if result == 1:
        return "RESERVED"       # Proceed to payment
    elif result == 0:
        return "SOLD_OUT"       # Tell user item is gone
    elif result == -1:
        return "ALREADY_BOUGHT" # Prevent duplicate purchases
```

### Phase 3: Request Queuing (Handling 1M concurrent requests)

Don't send all 1M requests to the inventory system. Use a queue + token system:

```python
import asyncio
import uuid

async def handle_purchase_request(user_id: str, sale_id: str):
    """
    HTTP handler — returns immediately with a token + polling URL.
    Actual processing is async via queue.
    """
    # Check if sale is open
    if not is_sale_open(sale_id):
        return {"status": "NOT_OPEN", "opens_at": get_open_time(sale_id)}
    
    # Bot protection: verify user passed CAPTCHA or queue position
    if not is_verified_user(user_id):
        return {"status": "VERIFICATION_REQUIRED"}
    
    # Add to processing queue — immediate response to user
    request_token = str(uuid.uuid4())
    queue.push({
        "request_token": request_token,
        "user_id": user_id,
        "sale_id": sale_id,
        "queued_at": time.time()
    })
    
    # Return polling token immediately (no waiting)
    return {
        "status": "QUEUED",
        "token": request_token,
        "poll_url": f"/sale/{sale_id}/status/{request_token}"
    }

async def poll_status(request_token: str) -> dict:
    """Client polls this every 2 seconds"""
    result = result_store.get(request_token)
    if not result:
        return {"status": "PROCESSING"}
    return result

# Worker processes queue FIFO:
async def worker(queue, redis_client):
    while True:
        job = await queue.pop()
        result = reserve_item(job["sale_id"], job["user_id"])
        
        if result == "RESERVED":
            # Generate short-lived payment URL (15 min to complete payment)
            payment_token = generate_payment_token(job["user_id"], job["sale_id"])
            result_store.set(job["request_token"], {
                "status": "SUCCESS",
                "payment_url": f"/checkout/{payment_token}",
                "expires_in": 900   # 15 minutes to complete payment
            }, ttl=3600)
        else:
            result_store.set(job["request_token"], {
                "status": result,  # SOLD_OUT or ALREADY_BOUGHT
            }, ttl=3600)
```

### Phase 4: Write-Back to Database (Async)

Redis is the fast path. The DB is the durable record:

```python
# After Redis reservation succeeds → async write to DB
async def persist_reservation(user_id: str, sale_id: str, payment_token: str):
    """
    Write reservation to DB asynchronously.
    Called after Redis reservation, before payment.
    """
    async with db.transaction():
        await db.execute("""
            INSERT INTO sale_reservations 
                (user_id, sale_id, payment_token, status, reserved_at, expires_at)
            VALUES 
                ($1, $2, $3, 'RESERVED', NOW(), NOW() + INTERVAL '15 minutes')
        """, user_id, sale_id, payment_token)

# Reservation expiry cleanup (background job):
async def release_expired_reservations():
    """
    Every minute: find expired reservations that weren't paid, release stock.
    """
    expired = await db.fetch("""
        SELECT user_id, sale_id FROM sale_reservations
        WHERE status = 'RESERVED' AND expires_at < NOW()
    """)
    
    for row in expired:
        # Release in Redis
        release_lua(keys=[f"sale:{row['sale_id']}:stock", 
                          f"sale:{row['sale_id']}:sold"],
                    args=[row['user_id']])
        
        # Update DB
        await db.execute("""
            UPDATE sale_reservations SET status = 'EXPIRED'
            WHERE user_id = $1 AND sale_id = $2 AND status = 'RESERVED'
        """, row['user_id'], row['sale_id'])
```

---

## Bot Protection

Flash sales are heavily targeted by scalper bots:

```
Layer 1: CAPTCHA / Proof-of-Work before purchase button activates
  → Adds 2-5 seconds of work; effective against simple bots

Layer 2: Virtual Queue (Waiting Room)
  → All users join waiting room 10 min before sale
  → Random shuffle → assigned queue positions
  → Fair: early arrival doesn't help bots
  Implemented by: Cloudflare Waiting Room, Queue-it, or custom Redis ZADD with random scores

Layer 3: Rate limiting per user
  → Max 1 purchase attempt per second per user_id
  → Implemented: Redis token bucket per user

Layer 4: Behavioral signals
  → Flag accounts created < 24h ago
  → Flag accounts with no previous purchase history
  → Require phone verification for flagged accounts

Layer 5: Device fingerprinting
  → Detect same device with multiple accounts
  → Canvas fingerprint, screen resolution, WebGL hash
```

---

## Key Design Decisions

| Decision | Naive | Flash-Sale Optimized |
|---|---|---|
| Stock storage | PostgreSQL row | Redis integer (atomic DECR) |
| Request handling | Synchronous DB write per request | Queue + async processing |
| User position | First-come-first-served on network | Virtual queue with randomization |
| Duplicate prevention | DB UNIQUE constraint | Redis SISMEMBER + SADD (atomic Lua) |
| Expired reservations | DB scan | Background job + Redis TTL |

---

## Scaling Challenges

| Challenge | Problem | Solution |
|---|---|---|
| Redis single point | 1M req/sec to one Redis node | Redis Cluster (shard by sale_id); stock operations stay on one shard |
| "Thundering herd" at T=0 | All 1M users hit at exactly the same microsecond | Jitter: sale opens at T+0 to T+2 sec randomly per user (perceived simultaneity, reduced spike) |
| Payment timeout wasted stock | User reserves but doesn't pay in 15 min | Expiry job releases stock back; TTL on payment tokens |
| Latency from queue | User doesn't know their position | Show estimated wait time based on queue depth and worker throughput |

---

## Hard-Learned Engineering Lessons

1. **Hitting the database directly**: saying "decrement the DB stock counter on each request" is the most common mistake. 1M concurrent DB writes will deadlock.

2. **Not using atomic operations**: doing `GET stock; if stock > 0: UPDATE stock = stock - 1` is a race condition. Two requests can both read stock=1 and both decrement, resulting in stock=-1 (oversell).

3. **Forgetting duplicate purchase prevention**: without checking if a user already bought, a user can make 1,000 requests and reserve 1,000 items.

4. **Not handling the async/queue model**: the correct design is: accept request immediately → queue → return token → client polls → worker processes queue against Redis. Synchronous checkout under 1M concurrent users doesn't work.

5. **Forgetting reservation expiry**: users who reserve but don't pay lock up stock. Must release reservations after 15 minutes.

6. **Not discussing bot prevention**: flash sales without bot mitigation result in bots buying all inventory within milliseconds. Virtual waiting rooms and CAPTCHA are essential production requirements for any high-demand sale system.

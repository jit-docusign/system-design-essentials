# Design an E-Commerce Checkout System (Amazon)

## Problem Statement

Design the checkout system for a large e-commerce platform. Users browse products, add items to a cart, then complete a purchase that requires inventory reservation, payment processing, order creation, and fulfillment. The system must handle thousands of concurrent checkouts, prevent overselling, and remain consistent even under failures.

**Scale requirements**:
- 300 million active users
- 50,000 orders per minute during peak (Prime Day, Black Friday)
- 500 million product listings
- 99.99% availability (< 52 minutes downtime per year)
- Idempotent: retrying a failed checkout must not double-charge

---

## Architecture Overview

```
User Browser/App
      │
      │ HTTPS
      ▼
┌─────────────────────────────────────────────────────────────────────┐
│  API Gateway + CDN (static product pages cached)                    │
└───────────────────────────┬─────────────────────────────────────────┘
                            │
        ┌───────────────────┼──────────────────────────┐
        │                   │                          │
        ▼                   ▼                          ▼
  ┌──────────┐      ┌──────────────┐           ┌──────────────┐
  │  Product │      │  Cart        │           │  Order       │
  │  Service │      │  Service     │           │  Service     │
  └──────────┘      └──────────────┘           └──────────────┘
        │                   │                          │
        │                   │                          │
  ┌─────┴─────┐      ┌──────┴───────┐          ┌───────┴──────┐
  │ Product   │      │  Redis Cart  │          │  Orders DB   │
  │ Catalog   │      │  (TTL 7d)    │          │  (PostgreSQL │
  │ (DynamoDB)│      │              │          │   sharded)   │
  └───────────┘      └──────────────┘          └──────────────┘
                                                       │
  ┌──────────────────┐  ┌───────────────────┐         │
  │ Inventory Service│  │ Payment Service   │◄────────┘
  │ (reservation +  │  │ (Stripe/Adyen     │ (saga orchestrator
  │  stock count)   │  │  integration)     │  calls these)
  └──────────────────┘  └───────────────────┘
```

---

## Key Domain Services

### 1. Cart Service

Cart is ephemeral state — no need for strong durability. Redis is ideal:

```python
import redis
import json
from datetime import timedelta

r = redis.Redis()
CART_TTL = timedelta(days=7)

def add_to_cart(user_id: str, product_id: str, quantity: int, price: float):
    key = f"cart:{user_id}"
    cart_item = {
        "product_id": product_id,
        "quantity": quantity,
        "price": price,           # Price at time of add-to-cart
        "added_at": time.time()
    }
    # Each item is a field in a hash: product_id → serialized item
    r.hset(key, product_id, json.dumps(cart_item))
    r.expire(key, CART_TTL)

def get_cart(user_id: str) -> list[dict]:
    key = f"cart:{user_id}"
    raw = r.hgetall(key)
    return [json.loads(v) for v in raw.values()]

def remove_from_cart(user_id: str, product_id: str):
    r.hdel(f"cart:{user_id}", product_id)

def clear_cart(user_id: str):
    r.delete(f"cart:{user_id}")

# Price validation at checkout time is critical:
# price in cart may be stale → always re-validate from Product Service before charging
```

### 2. Inventory Service — The Hard Part

Prevent overselling (selling more than available stock):

```python
# Stock management with Redis atomic operations
import redis

r = redis.Redis()

def initialize_stock(product_id: str, quantity: int):
    r.set(f"stock:{product_id}", quantity)

def reserve_stock(product_id: str, quantity: int) -> bool:
    """
    Atomically decrement stock. Return False if insufficient.
    Uses Lua script for atomic check-and-decrement.
    """
    lua_script = """
    local current = tonumber(redis.call('GET', KEYS[1]))
    if current == nil then
        return -1  -- product not found
    end
    if current < tonumber(ARGV[1]) then
        return 0   -- insufficient stock
    end
    redis.call('DECRBY', KEYS[1], ARGV[1])
    return 1       -- success
    """
    result = r.eval(lua_script, 1, f"stock:{product_id}", quantity)
    return result == 1

def release_stock(product_id: str, quantity: int):
    """Release stock if checkout is cancelled or payment fails"""
    r.incrby(f"stock:{product_id}", quantity)

# Database-level enforcement (belt-and-suspenders):
# Even with Redis reservation, write final deduction to PostgreSQL with:
# UPDATE inventory SET quantity = quantity - ? WHERE product_id = ? AND quantity >= ?
# AffectedRows == 0 → oversell detected → rollback
```

**Stock reservation vs. deduction**:
```
Time 0: stock = 100
User A adds to cart (no reservation yet) → stock still 100
User A starts checkout → RESERVE 1 → stock = 99 (held for 15 min)
User A completes payment → DEDUCT (finalize reservation) → permanent
User A times out checkout → RELEASE → stock = 100 again

Reservation TTL = 15 minutes: if payment not completed, auto-release
This prevents "ghost reservations" from abandoned carts
```

### 3. Order Service with Idempotency

Every checkout must be idempotent — safe to retry on network failure:

```python
import uuid
import hashlib

def create_checkout_session(user_id: str, cart_id: str) -> str:
    """
    Returns idempotency_key for this checkout attempt.
    Derived from user+cart so retries produce same key.
    """
    content = f"{user_id}:{cart_id}:{date.today()}"
    idempotency_key = hashlib.sha256(content.encode()).hexdigest()[:32]
    return idempotency_key

def process_checkout(user_id: str, idempotency_key: str, cart: list[dict]) -> dict:
    """
    Idempotent checkout: if same idempotency_key seen before, return cached result.
    """
    # Check cache first
    cached = idempotency_store.get(idempotency_key)
    if cached:
        return cached  # Return same result as before, don't re-process
    
    try:
        result = _execute_checkout(user_id, cart)
        idempotency_store.set(idempotency_key, result, ttl=86400)  # 24h cache
        return result
    except Exception as e:
        # Don't cache failures (allow retry)
        raise e

def _execute_checkout(user_id: str, cart: list[dict]) -> dict:
    order_id = str(uuid.uuid4())
    
    # Saga: sequence of steps, each with compensation
    steps_completed = []
    try:
        # 1. Validate prices (cart may have stale prices)
        validated_items = validate_prices(cart)
        total = sum(item["price"] * item["qty"] for item in validated_items)
        
        # 2. Reserve inventory
        for item in validated_items:
            reserve_stock(item["product_id"], item["qty"])
            steps_completed.append(("inventory", item))
        
        # 3. Create order record (pending payment)
        order = create_order(order_id, user_id, validated_items, total, status="PENDING")
        steps_completed.append(("order", order_id))
        
        # 4. Charge payment
        payment = charge_payment(user_id, total, order_id)
        steps_completed.append(("payment", payment["transaction_id"]))
        
        # 5. Confirm order
        update_order_status(order_id, "CONFIRMED")
        clear_cart(user_id)
        
        # 6. Emit event for fulfillment
        emit_event("order.confirmed", {"order_id": order_id, "items": validated_items})
        
        return {"order_id": order_id, "status": "CONFIRMED", "total": total}
    
    except Exception as e:
        # Compensate in reverse order
        compensate(steps_completed)
        raise e

def compensate(steps_completed: list):
    """Undo completed steps in reverse order (Saga compensation)"""
    for step_type, step_data in reversed(steps_completed):
        if step_type == "inventory":
            release_stock(step_data["product_id"], step_data["qty"])
        elif step_type == "payment":
            refund_payment(step_data)  # If payment succeeded but order failed
        elif step_type == "order":
            update_order_status(step_data, "CANCELLED")
```

### 4. Saga Pattern for Distributed Checkout

Single-machine transactions don't work across microservices. Use the Saga pattern:

```
Choreography Saga (event-driven):
─────────────────────────────────
checkout.initiated
    → Inventory Service hears event → reserves stock → emits inventory.reserved
    → Payment Service hears inventory.reserved → charges card → emits payment.captured
    → Order Service hears payment.captured → confirms order → emits order.confirmed
    → Fulfillment Service hears order.confirmed → starts picking

If payment fails:
    → Payment Service emits payment.failed
    → Inventory Service hears payment.failed → releases reserved stock
    → Order Service hears payment.failed → marks order FAILED

Orchestration Saga (centralized):
──────────────────────────────────
Order Orchestrator Service manages the entire flow:
  1. Call Inventory Service: reserve_stock() → OK/FAIL
  2. Call Payment Service: charge() → OK/FAIL
  3. If payment OK: call Order Service: confirm() 
  4. If payment FAIL: call Inventory Service: release_stock()

Orchestration is easier to reason about; preferred for checkout.
Choreography is more decoupled but harder to debug.
```

### 5. Payment Service Integration

```python
import stripe

class PaymentService:
    def charge(self, user_id: str, amount_cents: int, order_id: str) -> dict:
        """
        Two-phase payment: authorize first, capture after order confirmed.
        """
        # Get saved payment method
        customer = get_stripe_customer(user_id)
        
        # Phase 1: Authorization (hold funds, don't move money yet)
        payment_intent = stripe.PaymentIntent.create(
            amount=amount_cents,
            currency="usd",
            customer=customer.id,
            payment_method=customer.default_payment_method,
            confirm=True,
            capture_method="manual",  # Don't capture until order confirmed
            idempotency_key=f"order_{order_id}",  # Prevent double-charge
            metadata={"order_id": order_id}
        )
        
        if payment_intent.status != "requires_capture":
            raise PaymentAuthorizationFailed()
        
        return {
            "payment_intent_id": payment_intent.id,
            "status": "authorized",
            "amount": amount_cents
        }
    
    def capture(self, payment_intent_id: str) -> dict:
        """Phase 2: Capture (move money) after inventory confirmed"""
        payment_intent = stripe.PaymentIntent.capture(payment_intent_id)
        return {"status": "captured", "transaction_id": payment_intent.id}
    
    def cancel_authorization(self, payment_intent_id: str):
        """Release hold if inventory failed"""
        stripe.PaymentIntent.cancel(payment_intent_id)
```

---

## Order Data Model

```sql
CREATE TABLE orders (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         BIGINT NOT NULL,
    status          VARCHAR(20) NOT NULL,  -- PENDING|CONFIRMED|SHIPPED|DELIVERED|CANCELLED
    total_cents     INT NOT NULL,
    currency        VARCHAR(3) DEFAULT 'USD',
    shipping_address_id BIGINT REFERENCES addresses(id),
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    updated_at      TIMESTAMPTZ DEFAULT NOW(),
    idempotency_key VARCHAR(64) UNIQUE  -- prevents duplicate orders
);

CREATE TABLE order_items (
    id          BIGSERIAL PRIMARY KEY,
    order_id    UUID REFERENCES orders(id),
    product_id  VARCHAR(64) NOT NULL,
    seller_id   BIGINT NOT NULL,
    quantity    INT NOT NULL CHECK (quantity > 0),
    unit_price_cents INT NOT NULL,    -- locked at checkout time
    product_name TEXT NOT NULL,       -- snapshot at checkout (product may change)
    sku         VARCHAR(64)
);

CREATE TABLE order_events (
    id          BIGSERIAL PRIMARY KEY,
    order_id    UUID REFERENCES orders(id),
    event_type  VARCHAR(50),  -- CREATED, PAYMENT_CAPTURED, SHIPPED, etc.
    payload     JSONB,
    created_at  TIMESTAMPTZ DEFAULT NOW()
);

-- Shard orders by user_id (hash-based) for scale:
-- 300M users → shard across 16 DB nodes by user_id % 16
```

---

## Key Design Decisions

| Decision | Issue | Approach |
|---|---|---|
| Inventory consistency | Redis + DB must agree | Redis for fast check; DB as source of truth with `quantity >= 0` constraint |
| Payment before or after inventory? | Money moves but item unavailable | Auth first (hold only), capture after inventory confirmed |
| Cart storage | Redis TTL vs. DB | Redis (ephemeral, fast, survives page reload via TTL) |
| Oversell prevention | Two users buy last item simultaneously | Lua atomic decrement in Redis + SQL `CHECK quantity >= 0` |
| Saga vs 2PC | Distributed transaction across services | Saga with compensation (2PC blocks; Saga fails fast) |
| Price lock | Cart has stale price | Re-validate prices from Product Service at checkout time |

---

## Scaling Challenges

| Challenge | Problem | Solution |
|---|---|---|
| Flash sale (10,000 items → sold in 1 second) | Redis lock contention on hot items | Pre-compute "sold out" flag; place excess requests in queue |
| Order DB hot writes | 50,000 orders/min → single DB bottleneck | Shard by user_id; Kafka event stream for order events |
| Payment gateway timeouts | Stripe /charge times out after 10s | Store pending payment_intent; async webhook confirms payment |
| Fraud detection latency | Fraud check adds 200ms | Run fraud check async post-auth; cancel capture if fraud detected (< 30s) |
| Multi-seller orders | Cart has items from 3 sellers | Split into sub-orders per seller; each has its own fulfillment |

---

## Common Interview Mistakes

1. **Not addressing idempotency**: "What happens if the user clicks 'Buy' twice?" or "What if the network drops after payment succeeds but before order confirms?" — These require idempotency keys.

2. **Using distributed 2-Phase Commit**: 2PC across microservices creates tight coupling and blocking failures. Saga with compensation is the correct approach.

3. **Not distinguishing reservation from deduction**: "Reserve" holds stock during checkout window; "deduct" permanently removes it after payment. Not distinguishing these leads to oversell bugs.

4. **Forgetting compensation logic**: If payment fails after inventory reserves, you must release the reservation. The Saga compensation chain must be explicit.

5. **Charging before confirming inventory**: Charge the customer, then find out the item is out of stock → bad UX + refund overhead. Correct order: reserve inventory → authorize payment → capture.

6. **Not snapshotting prices and product names**: Product names change, prices change. Order records must copy the price and product description at checkout time, not store a reference.

7. **Using cart as the source of truth for prices**: Cart prices are advisory. Always re-fetch current price from Product Catalog at checkout time to prevent price manipulation.

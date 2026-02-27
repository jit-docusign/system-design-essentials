# Design Uber / Lyft (Ride-Sharing)

## Problem Statement

Design a ride-sharing platform where riders can request a ride, nearby drivers are matched, GPS positions are tracked in real-time, and pricing is calculated dynamically. Both parties track each other on a live map.

**Scale requirements (Uber)**:
- 5 million trips per day
- 2 million active drivers worldwide
- Peak: hundreds of thousands of simultaneous active trips
- Driver location update: every 4 seconds from each driver
- Matching must complete in < 3 seconds of ride request

---

## Core Features

1. **Rider requests a ride** — pickup location, destination, fare estimate
2. **Driver matching** — find nearest available driver
3. **Real-time GPS tracking** — rider and driver see each other on map
4. **Dynamic pricing (surge)** — demand/supply pricing in geographic areas
5. **Trip lifecycle** — accepted → en route → picked up → completed → rated

---

## High-Level Architecture

```
Rider App                    Backend                      Driver App
    │                           │                              │
    │── Request ride ──────────►│                              │
    │                           │                              │
    │                           │── Find nearby drivers ───────┤
    │                           │   (Driver Location Service)  │
    │                           │                              │
    │                           │── Dispatch Request ──────────►│
    │◄── Driver assigned ────── │◄── Driver accepts ────────── │
    │                           │                              │
    │◄── Driver location ──── ──│◄── GPS update (every 4s) ─── │
    │   (live map update)        │                              │
    │                           │                              │
    │◄── Trip completed ─────── │── Payment processed ─────────►│
```

---

## How it works

### 1. Driver Location Service (Core Challenge)

With 2 million drivers updating location every 4 seconds:

```
Write rate: 2,000,000 / 4 = 500,000 location updates/sec

Requirements for the location store:
  - Write-heavy (500K writes/sec)
  - Geospatial queries: "Find all drivers within 5 km of lat/lng"
  - Low latency reads (< 10ms for matching queries)
  - Data is ephemeral (stale after 8+ seconds without update)
```

**Storage: Redis with Geospatial Index**

```python
import redis

r = redis.Redis()

# Driver sends GPS update (every 4 seconds)
async def update_driver_location(driver_id: str, lat: float, lng: float):
    # GEOADD stores lat/lng in a Redis Sorted Set using geohash as score
    await r.geoadd('drivers:available', lat, lng, driver_id)
    
    # Set TTL on individual driver presence: auto-expire after 8 seconds
    # (2 missed updates = considered offline)
    await r.setex(f'driver:active:{driver_id}', 8, 1)
    
    # Remove if explicitly going offline
    # await r.zrem('drivers:available', driver_id)

# Find nearest available drivers
async def find_nearby_drivers(lat: float, lng: float, radius_km: float = 5.0) -> list:
    # GEORADIUS returns drivers within radius sorted by distance
    nearby = await r.geosearch(
        'drivers:available',
        longitude=lng,
        latitude=lat,
        radius=radius_km,
        unit='km',
        sort='ASC',
        count=10,  # Top 10 nearest
        withcoord=True,
        withdist=True
    )
    
    # Filter to only truly online drivers
    result = []
    for driver_id, dist_km, (driver_lng, driver_lat) in nearby:
        is_active = await r.exists(f'driver:active:{driver_id}')
        if is_active:
            result.append({'driver_id': driver_id, 'distance_km': dist_km})
    
    return result
```

**How Redis GEOADD works internally**:
```
Redis Geospatial = Sorted Set where score = 52-bit Geohash

GEOADD drivers:available 37.7749 -122.4194 "driver-xyz"
  → Encodes (lat, lng) as a Geohash integer
  → ZADD drivers:available <geohash_integer> "driver-xyz"

GEORADIUS / GEOSEARCH:
  → Convert center point to Geohash
  → Search neighboring Geohash cells within radius bounding box
  → Filter results by exact Haversine distance
```

### 2. Driver-Rider Matching (Dispatch)

```python
async def match_ride_request(request: RideRequest) -> Optional[str]:
    # Find nearby available drivers (within 5km)
    candidates = await find_nearby_drivers(
        request.pickup_lat, request.pickup_lng, radius_km=5.0
    )
    
    if not candidates:
        return None  # No drivers available
    
    # Rank candidates considering:
    # 1. Distance (primary: minimize ETA)
    # 2. Driver rating
    # 3. Acceptance rate (penalize drivers who reject frequently)
    # 4. Match with requested vehicle type (UberX vs Black, etc.)
    
    for candidate in rank_candidates(candidates, request):
        driver_id = candidate['driver_id']
        
        # Offer trip to driver (with 15-second timeout)
        accepted = await send_trip_offer(driver_id, request, timeout=15)
        
        if accepted:
            # Mark driver as busy (remove from available pool)
            await redis.zrem('drivers:available', driver_id)
            await redis.setex(f'driver:status:{driver_id}', 3600, 'on_trip')
            
            # Create trip record
            trip_id = await create_trip(request, driver_id)
            return trip_id
        else:
            continue  # Try next nearest driver
    
    return None  # All tried, none accepted
```

**Dispatch as a state machine**:
```
RIDE REQUEST LIFECYCLE:

Rider submits ride request
  → Status: SEARCHING
  → System offers to drivers sequentially by proximity
  
Driver accepts
  → Status: ACCEPTED
  → Driver navigating to pickup

Driver arrives at pickup
  → Status: DRIVER_ARRIVED

Rider gets in car, driver starts trip
  → Status: IN_PROGRESS

Driver arrives at destination
  → Status: COMPLETED
  → Payment triggered
  → Rating prompt

Transitions are stored in the trip record (MySQL) for auditability.
```

### 3. Real-Time Location Sharing During Trip

```
During an active trip, rider sees driver's live location:

Driver app → GPS update → Location Service
                                │
                           Trip Context Check:
                           "Is driver on active trip?"
                                │
                           YES: Broadcast to rider
                           via WebSocket
                                │
                         Rider app updates map marker
                         in real-time

Update frequency: every 4 seconds (GPS accuracy: ±3 meter)
Protocol: WebSocket (persistent connection during trip)

Alternative for iOS background: 
  APNS silent push (wake app) → GPS fetch → HTTP POST → WebSocket broadcast
```

### 4. Trip Route and ETA

```python
# ETA calculation using routing engine
async def compute_eta(driver_lat: float, driver_lng: float, 
                       dest_lat: float, dest_lng: float) -> int:
    # Option 1: Google Maps Distance Matrix API (external)
    # Option 2: Internal routing engine (Uber H3 grid on road graph)
    
    route = await routing_engine.route(
        origin=(driver_lat, driver_lng),
        destination=(dest_lat, dest_lng),
        depart_time=datetime.utcnow(),
        traffic_model='best_guess'  # current traffic conditions
    )
    
    return route.duration_seconds
```

Uber uses **H3** (Uber's hexagonal hierarchical grid system) to partition the globe into hexagonal cells of varying sizes. This is used for:
- Surge pricing zones (group supply/demand by hex cell)
- Driver supply heat maps
- ETA region-specific models

### 5. Surge Pricing

```python
async def compute_surge_multiplier(lat: float, lng: float) -> float:
    # Get H3 hex cell for this lat/lng
    h3_cell = h3.geo_to_h3(lat, lng, resolution=8)  # ~0.73 km² cells
    
    # Count demand (open requests in this cell in last 5 min)
    demand = await redis.get(f'demand:{h3_cell}') or 0
    
    # Count supply (available drivers) in this cell and neighbors
    neighboring_cells = h3.k_ring(h3_cell, k=2)   # cell + 2 rings
    supply = 0
    for cell in neighboring_cells:
        supply += len(await find_drivers_in_hex(cell))
    
    if supply == 0:
        return 4.0  # Maximum surge
    
    demand_supply_ratio = int(demand) / max(supply, 1)
    
    # Surge multiplier table
    if demand_supply_ratio < 1.0: return 1.0
    elif demand_supply_ratio < 1.5: return 1.2
    elif demand_supply_ratio < 2.5: return 1.5
    elif demand_supply_ratio < 4.0: return 2.0
    else: return min(demand_supply_ratio * 0.5, 4.0)  # Cap at 4x

# Demand counter: increment on ride request, decrement on match or timeout
async def on_ride_request(lat: float, lng: float):
    h3_cell = h3.geo_to_h3(lat, lng, resolution=8)
    ttl = 300  # 5-minute window
    await redis.incr(f'demand:{h3_cell}')
    await redis.expire(f'demand:{h3_cell}', ttl)
```

### 6. Payment and Fare Calculation

```python
async def calculate_fare(trip: Trip) -> decimal.Decimal:
    base_fare = Decimal('1.50')
    per_mile_rate = Decimal('1.25')
    per_minute_rate = Decimal('0.22')
    
    # Compute actual route distance and duration from GPS trace
    distance_miles = compute_trip_distance(trip.gps_trace)
    duration_minutes = (trip.ended_at - trip.started_at).seconds / 60
    
    fare = base_fare + (per_mile_rate * distance_miles) + (per_minute_rate * duration_minutes)
    fare = fare * trip.surge_multiplier
    fare = max(fare, Decimal('7.00'))  # Minimum fare
    
    return fare.quantize(Decimal('0.01'))
```

---

## Key Design Decisions

| Decision | Choice | Reasoning |
|---|---|---|
| Driver location store | Redis Geospatial (GEOADD/GEORADIUS) | 500K writes/sec; sub-10ms range queries |
| Location TTL | 8-second expiry per driver | Auto-clean offline drivers from available pool |
| Dispatch strategy | Sequential nearest → offer → timeout | Fair to drivers; avoids simultaneous offers |
| Location grid | H3 hexagonal grid | Uniform cell sizes; no edge distortion at poles |
| Live tracking | WebSocket (persistent during trip) | Push updates without polling overhead |
| Surge pricing | Demand/supply ratio per H3 cell | Geographic granularity; smooth adjustments |

---

## Scaling Challenges

| Challenge | Solution |
|---|---|
| 500K location updates/sec | Redis Geo: write to sorted set with Geohash score |
| Sub-3s matching in dense cities | In-memory location store; no DB queries in matching hot path |
| 2M drivers with 8s TTL | Redis SETEX per driver; automatic expiry without cleanup jobs |
| Trip state transitions at scale | MySQL (ACID: payment cannot double-charge); Redis for active trip state cache |
| Surge pricing by zone | H3 grid + Redis counters per hex cell; updated in real-time |

---

## Common Interview Mistakes

- **Using MySQL/PostgreSQL for location queries**: PostGIS can do geo queries, but 500K writes/sec with sub-10ms read latency is Redis territory.
- **Not handling driver going offline mid-search**: TTL-based expiry cleans this up automatically. Without TTL, you'd serve offline drivers to riders.
- **Dispatching to all nearby drivers simultaneously**: this causes the "thundering herd" — 10 drivers all simultaneously receive a request, and 9 reject when the first accepts. Sequential offer dispatch prevents this.
- **Not designing the trip state machine**: the lifecycle has many states and transitions. Missing state transitions means missing edge cases like driver no-shows, rider cancellations mid-trip, or app crashes.

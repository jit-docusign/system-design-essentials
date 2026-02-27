# Design Google Maps

## Problem Statement

Design a mapping and navigation service that displays maps, provides turn-by-turn navigation with live traffic, enables location search (places, addresses), and handles route computation globally.

**Scale requirements (Google Maps)**:
- 1 billion monthly active users
- 5 million businesses and landmarks indexed
- Satellite/street imagery: petabytes of tile data
- Real-time traffic from 1 billion+ location-reporting devices
- Route calculation in < 1 second
- Map tiles served in < 50ms globally

---

## Core Features

1. **Map display** — rendered from vector/raster tiles
2. **Search** — find places, addresses, businesses
3. **Navigation** — turn-by-turn routing with live traffic
4. **Live traffic** — crowd-sourced from users
5. **Street View / Satellite** — photographic layer

---

## High-Level Architecture

```
Mobile / Web Client
    │
    ├── Map Tiles ────────────────────────────────────► Tile CDN
    │   (displayed map background)                       │
    │                                                  Tile Storage (S3)
    ├── Place Search ────────────────────────────────► Search Service
    │                                                  (Elasticsearch)
    ├── Route Request ───────────────────────────────► Routing Service
    │   (origin, destination)                           (Graph algorithm)
    │                                                  Road Graph DB
    └── Location Update (navigation mode) ───────────► Traffic Service
                                                        (aggregation)
```

---

## How it works

### 1. Map Tiles

Maps are served as a grid of **tiles** — small square image (raster) or vector data chunks:

```
The world is divided into a tile pyramid:

Zoom 0: 1 tile covering entire world (256x256 pixels)
Zoom 1: 4 tiles (2x2 grid)
Zoom 2: 16 tiles (4x4 grid)
...
Zoom 18: 68 billion tiles (city block level detail)

Tile coordinate system:
  tile_z_x_y: zoom level, column, row
  e.g., tile_15_5241_12535 = Manhattan at zoom 15

Each tile shows:
  Roads, labels, water, parks (vector layers, rendered client-side)
  OR pre-rendered raster PNG/WebP images (older approach)
```

**Vector tiles** (modern approach — Google Maps switched to vector in 2013):
```
Advantages over raster tiles:
  - Client renders them at any zoom level smoothly (no pixelation)
  - Smaller data size (structured geometry vs. image pixels)
  - Labels can be rotated/localized without re-serving tiles
  - Road networks can be highlighted dynamically (traffic colors)
  
Format: Mapbox Vector Tile (MVT) — Protocol Buffer binary
  Contains: roads (lines), buildings (polygons), labels (points)
```

**Tile serving**:
```
Client requests tile at (z=15, x=5241, y=12535):
  1. Check local cache (client LRU cache, 512 MB on mobile)
  2. CDN edge node (nearest POP)
  3. CDN origin (pre-computed tile storage in GCS/S3)

Pre-computation:
  - All tiles up to zoom 12 pre-computed globally
  - Zoom 13–18 tiles computed or cached on-demand
  - Popular areas (cities) pre-warmed at all zoom levels
  - Rural areas: compute on first request, cache for 24h

Cache TTL:
  - Base map tiles: 30 days (streets don't change often)
  - Traffic tiles: 1–2 minutes (traffic changes constantly)
```

### 2. Place Search (Local Search)

```
User types "pizza near me" or "Empire State Building":
  
  1. Query parsed: intent (pizza = category) + location context (near me)
  
  2. Search index queried:
     Elasticsearch:
       - BM25 text match on name, category, address
       - Geo-filter: within X km of user's location
       - Ranking signals: rating, review count, distance, hours (open now?)
  
  3. Place data:
     Source: Google's Place Database
       - 200M+ locations: businesses, landmarks, addresses, transit
       - Updated from: Street View imagery, user contributions, business submissions
     
     Schema:
       {
         place_id: "ChIJN1t_tDeuEmsRUsoyG83frY4",
         name: "Empire State Building",
         lat: 40.7484405,
         lng: -73.9878531,
         categories: ["landmark", "tourist_attraction"],
         address: "350 5th Ave, New York, NY 10118",
         phone: "+1 212-736-3100",
         hours: {"mon": "10:00-22:00", ...},
         rating: 4.7,
         review_count: 85234
       }
```

### 3. Routing (Navigation) — The Core Algorithm

Given origin and destination, find the fastest / shortest route considering live traffic.

**Road network as a weighted directed graph**:
```
Nodes: road intersections
Edges: road segments (one-directional for one-way streets)
Edge weight: travel time = distance / speed (updated with live traffic)

Planet-scale graph:
  - ~100 million nodes (intersections)
  - ~500 million directed edges (road segments)
  - Doesn't fit in memory of a single machine
```

**Dijkstra's Algorithm** finds shortest path but is O((V + E) log V) — too slow for planet-scale:

```
For a continental route (New York → Los Angeles):
  Nodes: ~30 million in North America
  Dijkstra time: ~10 seconds

Google's approach: Contraction Hierarchies (CH)
  Preprocess the road graph:
    - Assign "importance" to each node (major highway > local street)
    - Create "shortcuts": precomputed paths that skip unimportant nodes
    - Queries search top of hierarchy first → zoom into local area near endpoints
    
  Query time with CH: ~50 milliseconds vs 10 seconds for Dijkstra
  Preprocessing time: ~1 hour (run offline, store result)
```

**Bidirectional A* with live traffic** (simplified):
```python
import heapq

def route(graph, start, end, heuristic, traffic_data):
    # A* forward from start + backward from end
    # Meet in the middle = drastically fewer nodes explored
    
    dist = {start: 0}
    priority_queue = [(heuristic(start, end), start)]
    
    while priority_queue:
        _, current = heapq.heappop(priority_queue)
        
        if current == end:
            return reconstruct_path(prev, start, end)
        
        for neighbor, edge in graph[current]:
            # Edge weight = base_time × traffic_multiplier
            traffic_factor = traffic_data.get((current, neighbor), 1.0)
            new_dist = dist[current] + edge.base_time * traffic_factor
            
            if neighbor not in dist or new_dist < dist[neighbor]:
                dist[neighbor] = new_dist
                h = heuristic(neighbor, end)
                heapq.heappush(priority_queue, (new_dist + h, neighbor))
    
    return None

# Heuristic: Haversine distance (straight-line)  
def heuristic(node, goal):
    return haversine_km(node.lat, node.lng, goal.lat, goal.lng) / 130  # fastest road speed
```

**Multiple route alternatives**:
```
Generate 3 routes:
  1. Fastest (default): minimize time with traffic
  2. No tolls: avoid toll roads (preference filter)
  3. Scenic / Less traffic: optimize for congestion avoidance

All three computed in < 1 second using CH + bidirectional search.
```

### 4. Live Traffic System (Crowd-Sourced)

```
Users in navigation mode send anonymized GPS pings:
  {lat, lng, bearing, speed, timestamp}
  Sent every 1 second while navigating.

Traffic Aggregation Service:
  1. Receive 1 billion+ GPS pings per day (10k+ per second)
  2. Map-match: "This GPS point is on segment X of road Y"
     (HMM: Hidden Markov Model for matching GPS noise to road network)
  3. Aggregate speeds by road segment:
     - Average speed last 5 minutes → current traffic
     - Historical speed for time of day/day of week → typical traffic
  4. Compute congestion levels:
     - Free flow: > 80% of speed limit → green
     - Slow: 40–80% → yellow  
     - Heavy: 20–40% → orange
     - Stop-and-go: < 20% → red

Traffic data freshness: updated every 2-3 minutes
Stored in: road segment lookup table (Redis for hot segments)
Applied to: routing weight recalculation, traffic tile color rendering
```

```python
# Map-matching: assign a GPS point to a road segment
from typing import Optional

def map_match_gps(lat: float, lng: float, bearing: float, 
                   speed: float, prev_segment: Optional[str]) -> str:
    # Find road segments within 30 meters
    nearby_segments = road_index.query_radius(lat, lng, radius_m=30)
    
    # Score each candidate:
    scores = {}
    for seg in nearby_segments:
        # 1. Distance to segment centerline
        dist_score = 1.0 / (1 + perpendicular_distance(lat, lng, seg))
        
        # 2. Heading alignment (GPS bearing matches road direction)
        heading_diff = abs(bearing - seg.bearing) % 360
        heading_diff = min(heading_diff, 360 - heading_diff)
        heading_score = 1.0 - heading_diff / 180.0
        
        # 3. Transition probability (if we were on segment X, can we be on Y now?)
        transition_score = transition_prob(prev_segment, seg.id)
        
        scores[seg.id] = dist_score * heading_score * transition_score
    
    return max(scores, key=scores.get)
```

### 5. Street View

```
Data collection:
  Google Maps cars with 360° camera rigs + GPS + LiDAR
  Cover roads globally; updated every 1-3 years in cities

Storage:
  360° panoramic images: ~50 GB per km driven
  Global coverage (tens of millions of km): exabytes of data
  Stored in Google Bigtable + GCS
  Displayed as 512×512 cubemap tiles (6 faces of a cube = full sphere)

Navigation:
  Graph of connected panoramas: {pano_id: [neighbor_pano_ids]}
  Each pano linked to its location on the road graph
  User "walks" = traverse the panorama graph
```

---

## Key Design Decisions

| Decision | Choice | Reasoning |
|---|---|---|
| Map format | Vector tiles (MVT) | Smaller, smooth zoom, dynamic styling |
| Tile CDN | CDN with long TTL for base, short for traffic | Road data stable; traffic volatile |
| Routing algorithm | Contraction Hierarchies + A* | Sub-second continental routing |
| Traffic aggregation | Crowd-sourced GPS + map-matching (HMM) | Fresh, real-world data from billions of devices |
| Place search | Elasticsearch + geo-filter | Full-text + proximity; flexible ranking |

---

## Scaling Challenges

| Challenge | Solution |
|---|---|
| Petabytes of tile data | GCS/S3 + CDN; pre-compute popular zoom levels |
| Route calculation for 1B users/day | Contraction Hierarchies preprocessing; ~50ms queries |
| 1B GPS pings/day for traffic | Kafka ingestion; Flink real-time aggregation per road segment |
| Map updates globally | Incremental tile regeneration on change; diff-based updates |
| 5M places search | Elasticsearch geo-queries; nearby top-K results |

---

## Common Interview Mistakes

- **Dijkstra for global routing**: pure Dijkstra is too slow at planet scale. Mention that production systems use precomputed hierarchies (Contraction Hierarchies or A* with heuristics).
- **Static traffic data**: live traffic is the critical differentiator for navigation apps. Crowd-sourced GPS → map-matching → segment speed aggregation is the pipeline to describe.
- **Raster tiles only**: modern maps use vector tiles for smooth rendering and dynamic styling. Mentioning the transition from raster to vector shows depth.
- **Not discussing tile caching hierarchy**: client cache → CDN → origin is essential for sub-50ms tile delivery at 1B users.

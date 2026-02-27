# Design Facebook News Feed

## Problem Statement

Design Facebook's news feed — the constantly updating list of posts (status updates, photos, videos, shares) from friends, pages, and groups a user follows, ranked by relevance rather than pure chronology.

**Scale requirements (Facebook)**:
- 3 billion users, 2 billion DAU
- News feed must load in < 100ms on first open
- 100 billion content interactions (likes, comments, shares) per day
- Each user follows hundreds of friends, pages, and groups
- Multi-content types: text posts, photos, videos, links, stories

---

## What's Different from Twitter

| Dimension | Facebook News Feed | Twitter Timeline |
|---|---|---|
| Content sources | Friends (bidirectional) + Pages + Groups | Followed accounts (unidirectional) |
| Ordering | ML-ranked relevance | Mostly reverse-chronological |
| Content types | Rich (video, links, events, groups) | Primarily text + media |
| Scale | 2B DAU | 200M DAU |
| Social graph | Dense (average 338 friends) | Sparse (asymmetric follow) |

---

## High-Level Architecture

```
Client
  │
  ▼
┌─────────────────────────────────────────────────────────────┐
│                       API Gateway                           │
└──────┬────────────────┬──────────────┬──────────────────────┘
       │                │              │
       ▼                ▼              ▼
Post Service      Feed Service     Social Graph
       │                │           Service
       ▼                │              │
   Post Store      Feed Retrieval      │
  (MySQL sharded)  – Redis cache       │
       │              – ML ranker      │
       │                │              │
       └── Fan-out ─────┘              │
           Service ────────────────────┘
```

---

## How it works

### Aggregation (Fan-out) Architecture

Facebook's feed uses a **multi-tier fan-out**:

```
User A posts → FeedAggregation Service
                   │
                   ├── Write to A's post store (MySQL sharded on user_id)
                   │
                   └── Publish to Kafka topic 'post-events'
                               │
                        Fan-out Consumer 
                               │
        ┌──────────────────────┼──────────────────────┐
        ▼                      ▼                      ▼
   Direct friends         Pages/Groups           Interest groups
   (fan-out to           (fan-out to             (recommendation
    friends' feeds)       subscribers)            candidates)
```

### Feed Retrieval — Ranked in Layers

Facebook's feed is not just a sorted list of posts — it goes through multiple ranking stages:

```
Request: "Give me user-789's news feed"
           │
           ▼
Step 1: Candidate Retrieval
  │  Source 1: Pre-computed fan-out feed (Redis) — recent posts from friends/pages
  │  Source 2: Groups the user is in (real-time pull)
  │  Source 3: Pages/Events (real-time pull)
  │  Source 4: Recommended content based on interests
  │  → ~5,000 candidate posts
           │
           ▼
Step 2: Filtering
  │  Remove: already-seen posts, violating content policy, blocked users
  │  → ~2,000 posts
           │
           ▼
Step 3: Relevance Scoring (ML Model — EdgeRank evolved)
  │  Features: relationship strength, post type affinity, recency, engagement velocity
  │  → Each remaining post gets a relevance score
           │
           ▼
Step 4: Post-Ranking Diversity
  │  Prevent showing 10 posts from the same person
  │  Mix content types (video, text, photo)
  │  Inject sponsored content at defined positions
  │  → ~25 posts for the first page
           │
           ▼
Render feed to client
```

```python
async def get_news_feed(user_id: str, page: int = 0) -> list:
    # Step 1: Candidate retrieval
    candidates = await gather_candidates(user_id)
    
    # Step 2: Filter
    seen_post_ids = await get_seen_posts(user_id)
    candidates = [c for c in candidates if c.id not in seen_post_ids]
    
    # Step 3: Score
    features = await extract_features(user_id, candidates)
    scores = await ranking_model.predict(features)
    scored = sorted(zip(candidates, scores), key=lambda x: -x[1])
    
    # Step 4: Diversity rerank
    feed = diversity_rerank(scored, max_per_author=4)
    
    # Track which posts were shown (for deduplication)
    shown_ids = [p.id for p in feed[:25]]
    await mark_posts_seen(user_id, shown_ids)
    
    return feed[:25]

async def gather_candidates(user_id: str) -> list:
    tasks = [
        get_friend_feed_candidates(user_id),   # from Redis fan-out
        get_group_posts(user_id),               # pull from group tables
        get_page_posts(user_id),                # pull from pages subscribed
        get_recommendation_candidates(user_id), # ML-based discovery
    ]
    results = await asyncio.gather(*tasks)
    return deduplicate(flatten(results))[:5000]
```

### Social Graph — The Critical Infrastructure

The Facebook social graph underpins all feed operations:

```
Connections:
  - Friendships (bidirectional, mutual)
  - Page follows (unidirectional)
  - Group memberships
  - Interest affinities (derived from engagement)

Scale:
  - 3B users × ~338 friends average = ~1T edges
  - Cannot fit in a single relational database

Solution: TAO (The Associations and Objects)
  - Custom graph store for Facebook's social graph
  - Objects: user, post, photo, event (nouns)
  - Associations: friend_of, liked_by, commented_on (verbs)
  - Geographically distributed read replicas
  - Write-through cache with MySQL backing
```

```python
# TAO-style interface (simplified)
tao = TAO()

# Create association
await tao.assoc_add(user_id, 'friend_of', friend_id, time=now())

# Read friends
friends = await tao.assoc_get(user_id, 'friend_of', limit=5000)

# Facebook's actual TAO is described in their 2013 USENIX paper
```

### Edgerank — Facebook's Ranking Algorithm

The original score formula (evolved significantly with ML since 2013):

$$\text{Score} = \sum_{\text{edges}} u_e \cdot w_e \cdot d_e$$

Where:
- $u_e$ = **Affinity**: how close are you with the content creator? (interaction history)
- $w_e$ = **Weight**: content type weight (video > photo > link > status)
- $d_e$ = **Decay**: time decay (newer posts score higher)

Modern Facebook feed uses a deep neural network with hundreds of features:
- User's interaction history with this author
- Post's engagement velocity in the first hour
- Device type (video autoplay on Wi-Fi vs. cellular)
- Time of day behavioral patterns
- Predicted probability of each action type (like, comment, share, hide, report)

### Handling Updates and Counts at Scale

```
Problem: 100B likes/day = >1M likes/sec
         Updating like_count column directly → hot row contention

Solution: Counter aggregation service
  1. Like event → Kafka
  2. Counter aggregation worker reads Kafka, batches by post_id
  3. Writes to Redis counter: INCR likes:{post_id}
  4. Async flush to MySQL: UPDATE posts SET like_count = like_count + delta WHERE id = ?
     (delta-based UPDATE, not SET, to handle race conditions)
  5. Cache serves like_count reads

Result: 1M+ like events/sec handled; MySQL sees ~1000 batched updates/sec
```

### Infinite Scroll + Cursor Pagination

```python
# Feed pagination using cursor (not offset-based which is non-deterministic with ranking)
class FeedCursor:
    def __init__(self, last_seen_timestamp: int, session_id: str):
        self.last_seen_timestamp = last_seen_timestamp
        self.session_id = session_id
        # session_id tracks a consistent "view" for this scroll session

async def get_feed_page(user_id: str, cursor: FeedCursor | None) -> dict:
    if not cursor:
        # First page: fresh ranking
        candidates = await gather_candidates(user_id)
        session_id = generate_session_id()
        await cache_ranked_feed(user_id, session_id, rank(candidates))
    else:
        # Continuation: use cached ranked feed for consistency
        candidates = await get_cached_ranked_feed(user_id, cursor.session_id)
    
    page = candidates[cursor_to_offset(cursor):cursor_to_offset(cursor) + 25]
    next_cursor = FeedCursor(page[-1].timestamp, cursor.session_id)
    
    return {"posts": page, "next_cursor": encode_cursor(next_cursor)}
```

---

## Key Design Decisions

| Decision | Choice | Reasoning |
|---|---|---|
| Feed ordering | ML-ranked, not chronological | Engagement optimization (2009 EdgeRank, now DNN) |
| Fan-out | Hybrid (fan-out for friends, pull for groups/pages) | Groups can have millions of members — can't fan-out |
| Social graph store | TAO (custom) | No off-the-shelf system handles FB's graph scale |
| Like counting | Redis + async flush | Avoid 1M/sec DB updates producing hot rows |
| Feed pagination | Session-tied cursor | Consistent scroll experience; prevents post duplication across pages |

---

## Scaling Challenges

| Challenge | Solution |
|---|---|
| 2B DAU timeline reads | Redis pre-computed feeds; CDN for static assets |
| 100B interactions/day | Kafka + counter aggregation workers |
| Trillion-edge social graph | TAO: custom distributed graph store |
| ML ranking at < 100ms | Two-stage: fast candidate retrieval → lightweight neural scorer |
| Groups with 1M+ members | Pull model for groups (not fan-out); async group feed cache |

---

## Hard-Learned Engineering Lessons

- **Forgetting groups and pages**: Facebook's feed isn't just friends. You must include groups and pages in your candidate retrieval design.
- **Chronological ordering**: Facebook does not sort by time. ML ranking is central to the product — at minimum, describe affinity × recency scoring.
- **Ignoring seen-posts deduplication**: infinite scroll without deduplication shows posts again on the next page. Maintain a per-session seen-post set.
- **No mention of ad injection**: real news feeds include sponsored content at defined positions. Even briefly mentioning the slot reservation mechanism shows depth.

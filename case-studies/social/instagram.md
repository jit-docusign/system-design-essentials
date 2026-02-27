# Design Instagram

## Problem Statement

Design a photo and video sharing platform where users can upload media, follow other users, and view a personalized feed of content from people they follow.

**Scale requirements (Instagram at peak)**:
- 1 billion users, 500 million DAU
- 100 million photos/videos uploaded per day
- 4.2 billion likes per day
- 50 billion photos stored
- Feed rendered in < 200ms

---

## Core Features

1. **Upload** photos/videos
2. **Follow** other users
3. **Feed** — see recent posts from people you follow
4. **Discover** — explore page, hashtags, search
5. **Notifications** — likes, comments, new followers

---

## High-Level Architecture

```
Mobile Client
     │
     ▼
┌─────────────────────────────────────────────────────────┐
│                         CDN                             │
│  (media delivery — images, videos, static assets)       │
└────────────────────┬────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────┐
│                   API Gateway                           │
│  (auth, rate limiting, routing)                         │
└──┬──────────────┬───────────────┬────────────────┬──────┘
   │              │               │                │
   ▼              ▼               ▼                ▼
Upload         Feed           User            Search
Service        Service        Service         Service
   │              │               │
   ▼              ▼               ▼
Object         Feed            User DB
Storage        Cache           (PostgreSQL)
(S3)           (Redis)
   │
   ▼
Media
Processing
(async)
```

---

## How it works

### 1. Photo Upload

The critical insight: **never write large files through your API servers**. Use pre-signed URLs for direct client-to-storage uploads.

```
Client                  Upload API             Object Storage (S3)
  │                         │                         │
  │── POST /upload ─────────►│                         │
  │   {filename, size, type} │                         │
  │                         │── generate presigned ───►│
  │                         │   PUT URL (15 min TTL)   │
  │◄── presigned_url ────────│                         │
  │                          │                         │
  │── PUT [presigned_url] ─────────────────────────────►│
  │   (upload directly)      │                         │
  │◄── 200 OK ─────────────────────────────────────────│
  │                          │                         │
  │── POST /upload/complete ─►│                         │
  │   {photo_id, storage_key}│                         │
  │                          │── publish to queue ─────►│ Media Processing Queue
  │◄── {photo_id, status} ───│                         │
```

```python
# Upload service — generate presigned URL
import boto3, uuid

s3 = boto3.client('s3')

async def initiate_upload(user_id: str, filename: str, content_type: str) -> dict:
    photo_id = str(uuid.uuid4())
    storage_key = f"photos/{user_id}/{photo_id}/{filename}"
    
    # Generate presigned URL (valid 15 minutes)
    presigned_url = s3.generate_presigned_url(
        'put_object',
        Params={'Bucket': 'instagram-uploads', 'Key': storage_key, 'ContentType': content_type},
        ExpiresIn=900
    )
    
    # Save pending photo record
    await db.photo.create({
        'id': photo_id,
        'user_id': user_id,
        'storage_key': storage_key,
        'status': 'pending',
        'created_at': datetime.utcnow()
    })
    
    return {'photo_id': photo_id, 'upload_url': presigned_url, 'storage_key': storage_key}

async def complete_upload(photo_id: str, user_id: str) -> None:
    # Publish to async media processing queue
    await queue.publish('media-processing', {
        'photo_id': photo_id,
        'user_id': user_id,
        'storage_key': photo.storage_key,
        'tasks': ['resize', 'thumbnail', 'blur-hash', 'moderate']
    })
    
    await db.photo.update(photo_id, {'status': 'processing'})
```

### 2. Media Processing (Async)

Multiple processing steps run in parallel after upload:

```
Raw Upload (S3)
      │
      ▼
Media Processing Workers (Kafka consumers)
  │
  ├── Thumbnail generator    (320x320, 160x160 for profile grid)
  ├── Multi-resolution resize (1080px, 750px, 640px, 480px)
  ├── HEIC → JPEG conversion
  ├── Blur hash computation  (placeholder while loading)
  ├── Content moderation     (ML model: NSFW detection)
  └── Metadata extraction    (EXIF: location, device)
      │
      ▼
Processed files written to CDN-backed S3 bucket
Photo record updated: status=published, cdn_urls=[...]
Fanout to followers' feeds triggered
```

### 3. Feed Generation — The Core Problem

The feed problem is the hardest part of Instagram at scale. There are two models:

**Fan-out on Write (Push model)** — write posts to all followers' feeds at write time:
```
User A (500 followers) posts → write to 500 feed caches immediately
User B's feed read = just read pre-computed feed from cache (fast)

Problem: celebrity with 100M followers posts → write 100M cache entries in seconds
```

**Fan-out on Read (Pull model)** — compute feed at read time:
```
User B opens feed → look up B's followed users → fetch recent posts from each → merge + rank
Fast writes, but slow reads (100+ DB queries per feed load)

Problem: doesn't scale for users following 1000+ accounts
```

**Instagram's hybrid approach (fan-out for normal users, pull for celebrities)**:

```python
CELEBRITY_FOLLOWER_THRESHOLD = 1_000_000

async def post_photo(user_id: str, photo_id: str) -> None:
    followers = await get_follower_count(user_id)
    
    if followers < CELEBRITY_FOLLOWER_THRESHOLD:
        # Fan-out on write: push to all followers' feed caches
        all_followers = await get_all_followers(user_id)
        
        # Batch write to feed cache in chunks
        for chunk in batch(all_followers, size=1000):
            await queue.publish('feed-fanout', {
                'photo_id': photo_id,
                'author_id': user_id,
                'timestamp': now(),
                'follower_chunk': chunk
            })
    # Celebrities: no fan-out, handled at read time

async def get_feed(user_id: str, cursor: str = None) -> list:
    # Step 1: Pull pre-computed feed from cache
    cached_feed = await redis.zrange(f'feed:{user_id}', 0, 30, withscores=True)
    
    # Step 2: For each followed celebrity, pull their latest posts
    followed_celebrities = await get_followed_celebrities(user_id)
    celebrity_posts = []
    for celeb_id in followed_celebrities:
        posts = await get_recent_posts(celeb_id, limit=5)
        celebrity_posts.extend(posts)
    
    # Step 3: Merge + sort + deduplicate
    merged = merge_sorted(cached_feed, celebrity_posts)
    
    # Step 4: Apply ranking (ML model for engagement prediction)
    ranked = await rank_feed(user_id, merged)
    
    return ranked[:20]
```

### 4. Feed Storage

```
Redis Sorted Set per user: key = "feed:{user_id}"
  Member: photo_id
  Score:  timestamp (unix milliseconds, with small engagement boost)

ZADD feed:user-456 1700000450000 "photo-123"
ZADD feed:user-456 1700001200000 "photo-789"
ZRANGE feed:user-456 0 30 REV WITHSCORES  → latest 31 posts

Max size: keep only last 800 posts per user
  → Run TTL or ZREMRANGEBYRANK to trim when > 800

Feed cache TTL: 7 days for active users; evict inactive users' feeds
```

### 5. Data Model

```sql
-- Users
CREATE TABLE users (
    id          UUID PRIMARY KEY,
    username    VARCHAR(30) UNIQUE NOT NULL,
    email       VARCHAR(255) UNIQUE NOT NULL,
    bio         TEXT,
    follower_count INT DEFAULT 0,
    following_count INT DEFAULT 0,
    created_at  TIMESTAMPTZ DEFAULT NOW()
);

-- Photos/Videos
CREATE TABLE media (
    id          UUID PRIMARY KEY,
    user_id     UUID REFERENCES users(id),
    caption     TEXT,
    cdn_urls    JSONB,       -- {"1080": "https://cdn.../1080.jpg", ...}
    blur_hash   VARCHAR(50),
    width       INT,
    height      INT,
    like_count  INT DEFAULT 0,
    comment_count INT DEFAULT 0,
    created_at  TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_media_user_created ON media(user_id, created_at DESC);

-- Follows (sharded by follower_id)
CREATE TABLE follows (
    follower_id UUID,
    followee_id UUID,
    created_at  TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (follower_id, followee_id)
);
CREATE INDEX idx_follows_followee ON follows(followee_id);  -- for fan-out

-- Likes (counter stored on media, this table for deduplication)
CREATE TABLE likes (
    user_id     UUID,
    media_id    UUID,
    created_at  TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (user_id, media_id)
);
```

### 6. Counting Likes at Scale

Updating `like_count` on every like causes contention on the `media` row at 4.2B likes/day:

```python
# Pattern 1: Redis counter with async DB flush
async def like_photo(user_id: str, photo_id: str) -> None:
    # Idempotency check (Bloom filter for quick rejection)
    already_liked = await likes_bloom.check(f"{user_id}:{photo_id}")
    if already_liked:
        return
    
    # Write to likes table (unique constraint prevents duplicates)
    try:
        await db.execute("INSERT INTO likes (user_id, media_id, created_at) VALUES ($1,$2,$3)",
                         user_id, photo_id, datetime.utcnow())
    except UniqueViolationError:
        return  # already liked
    
    # Increment Redis counter (fast, in-memory)
    await redis.incr(f"likes:{photo_id}")
    
    # Async: flush counters to DB every 30 seconds in background worker

async def get_like_count(photo_id: str) -> int:
    # Read from Redis (fast), fallback to DB
    count = await redis.get(f"likes:{photo_id}")
    return int(count) if count else await db.fetchval(
        "SELECT like_count FROM media WHERE id = $1", photo_id
    )
```

---

## Key Design Decisions

| Decision | Choice | Reasoning |
|---|---|---|
| Media storage | S3 + CDN (CloudFront) | Unlimited scale, global edge delivery |
| Feed model | Hybrid push/pull | Pure push breaks for celebrities; pure pull is slow |
| Like counters | Redis + async flush | Avoid DB contention at 50K likes/sec |
| Feed ranking | ML model (chronological is too simple) | Engagement optimization (Instagram 2016 shift) |
| Database | PostgreSQL (sharded) | Relational for follows/likes; shard by user_id |
| Search | Elasticsearch | Full-text on captions, hashtag search |

---

## Scaling Challenges

| Challenge | Solution |
|---|---|
| Celebrity fan-out (100M followers) | Pull model for celebrities only; hybrid threshold |
| Media storage (50B photos) | S3 with storage tiers (Standard → IA → Glacier by age) |
| Feed latency < 200ms | Pre-computed feed in Redis; async celebrity post fetch |
| 4.2B likes/day | Redis counters + async DB flush |
| Follow graph queries | Graph DB or denormalized adjacency list sharded by user |

---

## Hard-Learned Engineering Lessons

- **Only designing happy path**: discuss what happens when the media processing worker crashes mid-job (idempotent tasks, at-least-once delivery).
- **Forgetting the celebrity problem**: the fan-out model breaks badly for high-follower accounts. Always distinguish normal users from celebrities.
- **Single-region design**: discuss geographic replication — users in Europe should not route media requests to US data centers. CDN and multi-region object storage are required.
- **No mention of feed ranking**: Instagram moved away from chronological feed to ML ranking in 2016. Discuss how this changes the data pipeline.

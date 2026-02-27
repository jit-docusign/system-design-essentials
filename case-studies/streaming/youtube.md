# Design YouTube

## Problem Statement

Design a video sharing platform where users upload videos, which are processed and delivered globally, and viewers can search, discover, and watch content.

**Scale requirements (YouTube)**:
- 2.7 billion logged-in users per month; 122 million DAU
- 500 hours of video uploaded every minute
- 1 billion hours of video watched per day
- Average video resolution: 1080p → ~1 GB per raw hour
- Videos must start playing within 2 seconds of clicking

---

## Core Features

1. **Upload** videos (up to 12 hours / 256 GB)
2. **Transcode** to multiple resolutions
3. **Watch** with adaptive bitrate streaming
4. **Search** by title, description, transcript
5. **Recommendations** — home feed, up next
6. **Engagement** — likes, comments, subscriptions

---

## High-Level Architecture

```
Uploader                   Infrastructure                Viewer
    │                            │                          │
    │── Upload ──────────────────►│                          │
    │         (presigned URL)     │                          │
    │                         Object Storage                │
    │                         (S3 raw)                      │
    │                             │                          │
    │                         Transcoding                   │
    │                         Pipeline                      │
    │                         (Kafka + workers)             │
    │                             │                         │
    │                         CDN (multiple PoPs)          │
    │                             │◄── HTTPS HLS ──────────│
    │                             │    (adaptive stream)   │
    │                         Search Index                 │
    │                              │◄── search query ──────│
    │                         Recommendations              │
    │                              │◄── home feed ─────────│
```

---

## How it works

### 1. Video Upload Pipeline

Like TikTok, direct-to-storage upload via presigned URL:

```
Creator uploads video:
  1. Request presigned upload URL from Upload Service
  2. Client streams video directly to S3 (chunked upload with resumable support)
  3. Client calls /upload/complete
  4. Kafka event: {video_id, creator_id, storage_key, file_size}

Resumable upload (for large files / poor connectivity):
  - S3 multipart upload: split into 5–10 MB parts
  - Client tracks which parts are uploaded (local state)
  - On resume: skip already-uploaded parts
  - Parts assembled by S3 at the end
```

### 2. Transcoding Pipeline

YouTube's transcoding infrastructure is one of the largest in the world, processing 500 hours/min of video:

```
Raw video in S3
     │
     ▼
Kafka: video-uploaded event
     │
     ├── Video Transcoding Workers
     │   ┌─────────────────────────────────────────────┐
     │   │  Transcode to all resolutions:              │
     │   │  2160p (4K), 1440p, 1080p, 720p, 480p,     │
     │   │  360p, 240p, 144p (mobile/low bandwidth)    │
     │   │                                             │
     │   │  Format: VP9/AV1 (YouTube native), H.264   │
     │   │  Container: WebM, MP4                       │
     │   │  Segmented to 10-second HLS/DASH chunks     │
     │   └─────────────────────────────────────────────┘
     │
     ├── Thumbnail Generator
     │   → Extract 3 frames at 25%, 50%, 75% of duration
     │   → AI thumbnail selection
     │   → Creator custom thumbnail upload
     │
     ├── Audio Track Extraction
     │   → Separate audio track for dubbing/translation
     │   → Automatic captions (speech-to-text)
     │
     ├── Content Moderation
     │   → Policy violations: violence, copyright, spam
     │   → Video fingerprint check (Content ID system)
     │
     └── Metadata Extraction + Feature Vectors
         → Scene classification, object detection
         → Features for recommendation engine
         → Stored in Feature Store
```

**Parallel vs Sequential Transcoding**:
Different resolutions can be transcoded in parallel. YouTube uses a **divide and conquer** approach — a single video is split into segments, each segment transcoded in parallel by different machines, then assembled:

```
Video: 1 hour long = 360 × 10-second segments

Sequential:   1 machine × 60 seconds per segment = 360 minutes total
Parallel:     360 machines × 60 seconds = 1 minute total
(then assemble HLS playlist)
```

### 3. Adaptive Bitrate Streaming (ABR)

```
master.m3u8 (served from CDN):
  #EXT-X-STREAM-INF:BANDWIDTH=400000,RESOLUTION=640x360
  360p/index.m3u8
  #EXT-X-STREAM-INF:BANDWIDTH=1500000,RESOLUTION=1280x720
  720p/index.m3u8
  #EXT-X-STREAM-INF:BANDWIDTH=4000000,RESOLUTION=1920x1080
  1080p/index.m3u8

Player behavior:
  1. Fetch master.m3u8
  2. Start with 360p while measuring bandwidth (buffer-based ABR)
  3. If 1000ms segments download in < 800ms → sufficient bandwidth → switch up
  4. If downloads stall → switch down
  5. Buffer minimum 30 seconds ahead
```

### 4. CDN Architecture — Global Delivery

```
                         ┌─────────────────────────┐
                         │  Google's own CDN        │
                         │  (Google Edge Network)   │
                         │                         │
                         │  NYC PoP                │
                         │  London PoP             │
                         │  Tokyo PoP              │
                         │  São Paulo PoP           │
                         │  ... (100+ PoPs)         │
                         └─────────────────────────┘

Video files served from nearest PoP:
  - Popular videos: cached at all PoPs (proactively pushed)
  - Less popular: cached only at PoPs where they're requested (pull-through)
  - Cache hit ratio for popular content: > 99%

Video popularity follows a power law:
  Top 20% of videos account for 80% of views
  → Pre-warm CDN with top 20% — massive bandwidth savings
```

### 5. Video Metadata Storage

```sql
-- Videos: sharded by video_id or creator_id
CREATE TABLE videos (
    id              UUID PRIMARY KEY,
    creator_id      UUID,
    title           VARCHAR(100),
    description     TEXT,
    duration_secs   INT,
    view_count      BIGINT DEFAULT 0,
    like_count      BIGINT DEFAULT 0,
    comment_count   BIGINT DEFAULT 0,
    status          VARCHAR(20),  -- processing, published, removed
    manifest_url    TEXT,         -- CDN URL to master.m3u8
    thumbnail_url   TEXT,
    category        VARCHAR(50),
    tags            TEXT[],
    published_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_videos_creator ON videos(creator_id, published_at DESC);
CREATE INDEX idx_videos_category ON videos(category, published_at DESC);
```

**View count at scale (1B views/day = ~12K/sec)**:
```python
# Don't update DB on every view — use Redis + batch flush
async def increment_view(video_id: str, user_id: str):
    # Approximate dedup: count once per user per 24h using Redis set
    key = f"views:24h:{video_id}"
    already_counted = await redis.sismember(key, user_id)
    if not already_counted:
        await redis.sadd(key, user_id)
        await redis.expire(key, 86400)
        await redis.incr(f"view_counter:{video_id}")
    
# Background worker: flush every 30 seconds
async def flush_view_counts():
    keys = await redis.keys("view_counter:*")
    for key in keys:
        video_id = key.split(":")[1]
        count = await redis.getdel(key)
        if count:
            await db.execute(
                "UPDATE videos SET view_count = view_count + ? WHERE id = ?",
                int(count), video_id
            )
```

### 6. Search

```
Upload → Kafka → Search Indexer → Elasticsearch
  
Index per video:
  {
    "video_id": "dQw4w9WgXcQ",
    "title": "Never Gonna Give You Up",
    "description": "Rick Astley's official music video",
    "transcript": "We're no strangers to love you know the rules...",
    "tags": ["music", "80s", "pop"],
    "category": "Music",
    "view_count": 1500000000,
    "published_at": "1987-07-27T00:00:00Z"
  }

Query: "rick astley dance"
  → BM25 text match on title, description, transcript
  → Boost recent + popular videos
  → Personalization: boost videos from subscribed channels
```

### 7. Recommendations

YouTube's recommendation is based on a two-stage neural network:

```
Stage 1: Candidate Generation
  Input: user's watch history, liked videos, subscriptions
  Model: Two-tower NN (user embedding + video embedding)
  Output: ~200 candidate videos

Stage 2: Ranking
  Features per video:
    - Video quality signals (CTR, watch time %, likes/views ratio)
    - User-video match (embedding similarity)
    - Freshness (recent videos boosted)
    - Diversity (avoid all-same-channel results)
  Output: Final ranked list of ~20 videos for home feed
```

---

## Key Design Decisions

| Decision | Choice | Reasoning |
|---|---|---|
| Storage format | HLS/DASH adaptive bitrate | Works across all devices and network conditions |
| Transcoding | Parallel segment-level parallelism | Reduces transcoding time from hours to minutes |
| CDN strategy | Power-law aware: pre-warm top 20% | Cost efficiency; 99%+ cache hit on popular content |
| View counting | Redis buffer + async flush | Handle 12K/sec without DB contention |
| Recommendation | Two-stage NN (candidate + ranker) | Accuracy at scale; separates recall from ranking |

---

## Scaling Challenges

| Challenge | Solution |
|---|---|
| 500 hrs/min upload → transcode | Massive parallel transcoding on segment level |
| 1B views/day video delivery | CDN with smart pre-warming based on popularity |
| 1B views/day view count increment | Redis counter buffer; batch flush to DB |
| Search across billions of videos | Elasticsearch with incremental index updates via Kafka |
| First-play latency < 2s | CDN pre-warms first segments; ABR starts at low quality immediately |

---

## Hard-Learned Engineering Lessons

- **Single-quality video storage**: always design for adaptive bitrate (multiple resolutions). A single 1080p file doesn't work for mobile users on 3G.
- **Not mentioning the CDN pre-warming strategy**: "use a CDN" is obvious. The insight is proactively warming popular videos vs. pull-through caching for tail content.
- **Ignoring transcoding parallelism**: explaining how 500 hrs/min of video gets transcoded requires discussing segment-level parallel processing.
- **Synchronous view count DB updates**: 12,000 increments per second on a single column causes deadlocks. Always mention the Redis buffer + async flush pattern.

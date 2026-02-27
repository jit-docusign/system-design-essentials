# Design TikTok (Short Video Platform)

## Problem Statement

Design a short-form video platform where users upload videos (15 seconds to 10 minutes), and a recommendation engine serves a personalized "For You Page" (FYP) of videos with very high engagement and low latency.

**Scale requirements (TikTok)**:
- 1 billion users, 600 million DAU
- 500 million videos uploaded per day
- 1 billion videos served per day
- FYP loads first video within 500ms of app open
- Video starts playing within 200ms of swipe

---

## What Makes TikTok Unique

| Dimension | TikTok | Instagram / YouTube |
|---|---|---|
| Discovery model | Interest graph (strangers' content) | Social graph (who you follow) |
| Feed type | Single full-screen video, infinite swipe | Grid / subscription list |
| Watch signal | Watch time %, completion rate, replays | Likes, comments |
| Cold start | New user gets good content on Day 1 | Requires following people first |
| Creator strategy | Viral reach for unknown creators | Network effect favors established creators |

The "For You Page" works without a social graph — TikTok's core innovation over Instagram and YouTube.

---

## High-Level Architecture

```
Mobile Client
    │
    ├── Upload flow
    │      └──► Upload Service ──► Object Storage ──► Video Processing Pipeline
    │
    └── Feed (FYP) flow
           └──► Feed Service ──► Recommendation Engine ──► Video CDN (pre-buffered)
```

```
┌────────────────────────────────────────────────────────────┐
│                       API Gateway                          │
│  (auth JWT, rate limiting, geo routing)                    │
└──┬──────────────┬──────────────┬───────────────────────────┘
   │              │              │
   ▼              ▼              ▼
Upload          Feed           User/Social
Service         Service        Service
   │              │              │
   ▼              ▼              ▼
Object        Recommendation  User DB
Storage (S3)  Engine          (MySQL)
   │              │
   ▼              ├── Candidate Generator
Video             └── Ranker (ML model)
Processing
Pipeline
(Kafka workers)
```

---

## How it works

### 1. Video Upload Pipeline

```
Mobile client
     │
     ▼
Upload Service
     │── Generate presigned PUT URL ──► S3 (raw upload bucket)
     │
     │── Client uploads directly to S3 presigned URL
     │
     └── Client calls /upload/complete → triggers async pipeline

Raw Video in S3
     │
     ▼
Kafka: video-uploaded event
     │
     ├── Transcoding Worker
     │     └── Transcode to adaptive bitrate (HLS / DASH):
     │           144p, 360p, 720p, 1080p
     │           Each resolution → 2-10 second segments (HLS chunks)
     │
     ├── Thumbnail Generator
     │     └── Extract 3 thumbnail frames, generate animated preview
     │
     ├── Audio Processing
     │     └── Extract audio track, normalize, detect original sound
     │
     ├── Content Moderation
     │     └── ML model: violence, nudity, policy violations → auto-remove or flag
     │
     └── Metadata Extraction + Feature Computation
           └── Scene classification, object detection, sound category
               → Features used by recommendation engine
               → Store in Feature Store (Redis + Hive)
```

### 2. Adaptive Bitrate Streaming (HLS)

TikTok serves video as HLS (HTTP Live Streaming) segments from CDN:

```
video-xyz/
  master.m3u8          ← Manifest: lists all quality variants
  360p/
    index.m3u8         ← Playlist for 360p
    seg_000.ts         ← 2-second segment 0
    seg_001.ts
    ...
  720p/
    index.m3u8
    seg_000.ts
    ...

Client behavior:
  1. App opens → fetch master.m3u8 from CDN
  2. Start with 360p (fast start) while downloading 720p
  3. Network bandwidth monitor → upgrade to 1080p if bandwidth allows
  4. Pre-buffer next 1-2 videos in the background
```

```
CDN Pre-warming:
  Recommendation engine predicts next likely video
  → Pre-warm CDN edge nodes in user's region with that video's first 3 segments
  → When user swipes, first segment is already at nearest edge (< 50ms first byte)
```

### 3. For You Page (FYP) — Recommendation

The FYP is the crown jewel of TikTok. It recommends content from creators the user has **never followed** based purely on observed behavior.

**Two-stage retrieval + ranking**:

```
User opens FYP → Feed Service
                       │
            ┌──────────▼──────────────┐
            │  Stage 1: Candidate     │
            │  Retrieval (~10K videos)│
            │                         │
            │  - Collaborative filter │
            │    (users like you also │
            │    liked these videos)  │
            │  - Content-based filter │
            │    (videos similar to   │
            │    ones you watched)    │
            │  - Trending in region   │
            │  - Following creators   │
            └──────────┬──────────────┘
                       │
            ┌──────────▼──────────────┐
            │  Stage 2: Ranking        │
            │  (~10K → 20 videos)      │
            │                          │
            │  ML model predicts:      │
            │  - P(complete watch)     │
            │  - P(like)               │
            │  - P(share)              │
            │  - P(follow creator)     │
            │  Weighted score →        │
            │  Top 20 ranked           │
            └──────────┬──────────────┘
                       │
                 20 videos served
                 (next 5 pre-fetched)
```

**User interaction signals used by the ranker**:

| Signal | Weight | Meaning |
|---|---|---|
| **Watch time %** | Very high | Watched 90% = strong positive signal |
| **Replay** | Very high | Watched again = loved it |
| **Like** | High | Explicit positive |
| **Share** | Very high | Highest engagement intent |
| **Follow creator** | Medium | Interest in creator, not just this video |
| **Comment** | High | Engaged enough to write |
| **Skip < 2 sec** | Very negative | Not interesting at all |
| **Scroll away** | Negative | Stopped watching early |
| **Report** | Very negative | Punishes the video and similar content |

```python
# Recommendation score (simplified concept)
def score_video(user_id: str, video_id: str) -> float:
    user_features = feature_store.get_user_features(user_id)
    video_features = feature_store.get_video_features(video_id)
    
    # Features: user's interest categories, watch history stats,
    # video's engagement rates, category, audio, creator stats
    features = {**user_features, **video_features}
    
    # Model outputs probabilities for multiple actions
    p_complete = model.predict(features, target='complete_watch')
    p_like = model.predict(features, target='like')
    p_share = model.predict(features, target='share')
    
    # Weighted combination
    score = 0.4 * p_complete + 0.3 * p_share + 0.3 * p_like
    return score
```

**Cold Start for New Users**:

```
New user installs TikTok
  │
  ▼
Step 1: Show 5 videos from top trending (locale-specific)
  │     → Capture initial watch signals
  │
  ▼
Step 2: After 3 completed watches, infer rough interests
  │     → "User watched 2 cooking videos and 1 pet video"
  │
  ▼
Step 3: Switch to content-based candidate retrieval
        → Similar videos to completed watches
        → Collaborative filter bootstrapped from similar users
```

**Cold Start for New Videos**:

```
New video uploaded
  │
  ▼
Initial Exposure Pool: show to ~100 users (diverse demographics)
  │
  ▼
If positive engagement (60%+ completion rate, like rate > avg):
    → Expand to 1,000 users
    → If still positive → 10,000 → 100,000 → viral
    → Each expansion triggers a re-scoring in the ranker

If negative engagement (< 30% completion):
    → Limit distribution (not punished permanently, but not boosted)
```

### 4. Video Storage and CDN Architecture

```
Upload Bucket (S3)         Processed Bucket (S3)
  (raw originals)            (HLS segments per quality)
       │                            │
       │                            ▼
       │                     CDN (CloudFront or Akamai)
       │                     ┌───────────────────────────┐
       │                     │ Edge 1: Tokyo             │
       │                     │ Edge 2: Singapore         │
       │                     │ Edge 3: Mumbai            │
       │                     │ Edge 4: Frankfurt         │
       │                     │ Edge 5: New York          │
       │                     └───────────────────────────┘
       │
  Originals kept indefinitely (S3 Standard → IA after 90 days)
  Processed HLS segments: kept as long as video is live
```

**Pre-fetching strategy** (critical for low-latency swipe):

```python
# Client-side: after loading video N, start prefetching N+1, N+2
async def prefetch_next_videos(feed: list, current_index: int):
    for i in range(current_index + 1, min(current_index + 3, len(feed))):
        video_id = feed[i]['id']
        manifest_url = feed[i]['manifest_url']  # CDN URL for master.m3u8
        
        # Pre-download first 3 HLS segments of 360p into local cache
        segments = await fetch_manifest_segments(manifest_url, quality='360p', count=3)
        await client_cache.store(video_id, segments)

# Result: when user swipes, first segment is local → play in < 50ms
```

---

## Key Design Decisions

| Decision | Choice | Reasoning |
|---|---|---|
| Discovery model | Interest graph (not social graph) | Viral reach for unknown creators; better Day 1 UX |
| Video format | HLS adaptive bitrate | Works on all networks; graceful quality degradation |
| CDN strategy | Pre-fetch next 2-3 videos to edge | Makes swipe latency feel instant |
| Cold start | ~100 user exposure → gradual expansion | Virality potential for every video, not just established creators |
| Candidate retrieval | Collaborative filter + content-based | Two-tower model for fast approximate nearest neighbor |

---

## Scaling Challenges

| Challenge | Solution |
|---|---|
| 500M video uploads/day | Pre-signed PUT → S3 → async processing pipeline (Kafka workers) |
| 1B videos served/day | CDN with pre-warming; HLS adaptive streaming |
| FYP < 500ms cold start | Two-stage retrieval: fast candidate retrieval (ANN) + batch ranker |
| Pre-fetching 600M users × 2-3 videos | Predicted-next-video CDN warming; regional edge nodes |
| New video cold start fairness | Controlled exposure pools; expansion based on engagement thresholds |

---

## Hard-Learned Engineering Lessons

- **Designing a social-graph feed**: TikTok's FYP is interest-graph based. Designing it like Twitter (fan-out from follows) misses the core product insight.
- **Forgetting the two-stage recommendation**: 10K candidates → ML ranking → 20 results. Ranking 1M videos in real-time is not feasible; you need a fast candidate retrieval first.
- **Not addressing video delivery latency**: how video gets to the screen in < 200ms after a swipe is a design problem (pre-fetching, CDN edge warming, HLS chunking). Don't just say "serve from CDN."
- **Ignoring cold start for both users and videos**: new user and new video cold start problems require distinct solutions.

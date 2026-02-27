# Design Spotify

## Problem Statement

Design a music streaming platform where users can stream songs on demand, create playlists, discover music through algorithmic recommendations, and listen offline.

**Scale requirements (Spotify)**:
- 600 million users, 250 million DAU
- 100 million+ songs in the catalog
- 4 billion playlist creates; 2.8 billion playlist-hours streamed per month
- Audio stream must start within 500ms
- Offline mode: sync up to 10,000 songs per device
- Low latency for radio/continuous play (must queue next track before current ends)

---

## What Makes Spotify Unique

| Dimension | Spotify | Netflix/YouTube |
|---|---|---|
| Content type | Audio (small files: 3–10 MB per song) | Video (large: GB per film) |
| Offline mode | First-class (download up to 10K songs) | Offline is a premium feature, limited |
| Recommendation | Discover Weekly, Daily Mix, Radio (world-class) | Home feed, Up Next |
| Catalog freshness | 60,000 new songs/day | New titles weekly |
| Social features | Shared playlists, collaborative, friend activity | Minimal |
| Real-time features | Spotify Connect (same session across devices) | No multi-device sync |

---

## High-Level Architecture

```
Mobile / Desktop Client
     │
     ├── Browse/Search/Discover ──────────────────► API Services
     │   (catalog, playlists, recommendations)            │
     │                                                  Spotify
     ├── Stream song ────────────────────────────────► CDN (Akamai / Fastly)
     │   (OGG Vorbis or AAC, multiple bitrates)            │
     │                                                Origin (S3 + GCS)
     └── Download for offline ───────────────────────────►│
         (encrypted local cache)
```

---

## How it works

### 1. Audio Encoding and Storage

Audio files are much smaller than video — a 3-minute song at 320kbps is ~7 MB:

```
Label submits master audio (WAV/FLAC, lossless)
     │
     ▼
Ingestion Pipeline:
  Three quality tiers (OGG Vorbis):
    - 24 kbps  (very low bandwidth, restricted)
    - 96 kbps  (free tier mobile, low bandwidth)
    - 160 kbps (free tier desktop, standard)
    - 320 kbps (premium tier)
  
  AAC for Apple devices (Safari WebKit):
    - 128 kbps, 256 kbps

  Same file is served to all users at the quality tier their plan allows.
  DRM is applied (FairPlay, Widevine, PlayReady) for premium content.
```

### 2. Audio Streaming

Unlike video, audio doesn't need adaptive bitrate streaming (files are small enough to buffer quickly):

```
User clicks "Play" on a song:
  1. Client requests track_url from Spotify API
     → API checks: valid subscription, DRM license
     → Returns CDN URL + DRM token (Widevine/FairPlay) 

  2. Client fetches audio from CDN (~7 MB)
     → Start playback after 100–200 KB in buffer (~0.5 seconds at 320kbps)
     → Buffer rest of song while playing

  3. Client pre-fetches next track while current is playing
     → When current song is at 75% → start fetching next song
     → Ensures gapless playback (<100ms gap)

Pre-fetching logic:
  Queue position > 75% complete:
      prefetch(queue[current_index + 1])  # next song
  Queue position > 90%:
      prefetch(queue[current_index + 2])  # next-next song
```

### 3. Offline Mode

Premium users can download up to 10,000 songs per device:

```
Sync flow:
  1. User marks playlist as "Available Offline"
  2. Spotify app queues download jobs for all tracks in playlist
  3. Each track:
     a. Request download URL (signed, time-limited CDN URL)
     b. Download encrypted audio file
     c. Store encrypted file in local cache
     d. Store DRM license (time-limited validity)

  Local storage:
    - Encrypted with device-specific key + Spotify DRM key
    - Cannot be transferred to another device / played outside Spotify

License refresh:
  - DRM licenses expire after 30 days offline
  - App requires internet connection at least once per 30 days to refresh
  - Tracks are "deactivated" if license expires and no connectivity

Storage management:
  - Max 10,000 songs (~70 GB at 320kbps)
  - LRU eviction: oldest downloads removed when storage limit reached
  - User can manually remove specific downloads
```

### 4. Metadata and Catalog

```sql
-- Tracks (100M+ items, queried billions of times/day)
-- Stored in Cassandra for read scale
tracks:
  track_id UUID PK
  title TEXT
  artist_ids LIST<UUID>
  album_id UUID
  duration_ms INT
  language TEXT
  audio_features JSONB  -- tempo, energy, valence, danceability (for recommendation)
  popularity_score FLOAT  -- updated daily from streaming counts
  cdn_urls JSONB  -- {"96kbps": "...", "160kbps": "...", "320kbps": "..."}

-- Artists, Albums (MySQL for complex queries)
artists:
  artist_id UUID PK
  name TEXT
  genres TEXT[]
  popularity_score FLOAT
  monthly_listeners BIGINT

-- Playlists (Cassandra, write-heavy)
playlists:
  playlist_id UUID
  owner_user_id UUID
  title TEXT
  track_ids LIST<UUID>   -- ordered list of tracks
  is_collaborative BOOL
  follower_count INT

-- User playlists library
user_playlist_library:
  user_id UUID
  playlist_id UUID
  added_at TIMESTAMPTZ
  PRIMARY KEY (user_id, added_at, playlist_id)
```

### 5. Recommendations — Discover Weekly and Daily Mix

Spotify's recommendation is considered one of the best in the industry:

```
Data sources used:
  1. Collaborative Filtering:
     "Users who like what you like also like X"
     - Matrix factorization (implicit feedback: play counts, skips, saves)
     - Trained on billions of streaming events weekly

  2. Content-Based (NLP on playlists):
     - Analyze text of playlists: if every playlist named "chill studying" 
       includes this artist → artist is associated with that context
     - Apply word2vec to playlist names + track names

  3. Audio Features (Echo Nest / Spotify internally):
     Each track has computed features:
     {tempo, energy, valence, danceability, acousticness, speechiness, instrumentalness, liveness}
     → Used for "More like this" and Radio recommendations

  4. Taste Profile:
     User's explicit actions: liked songs, saved albums, followed artists
     + Implicit: play-through rate, skip rate, add to playlist

Discover Weekly (generated every Monday):
  1. Generate 30-song playlist per user (~200M users × 30 songs = 6B song lookups)
  2. Constraint: no tracks user has heard in last 30 days
  3. Constraint: no tracks user has already liked/saved
  4. Optimized for max novelty + max predicted enjoyment
  5. Pre-generated Saturday night (off-peak) → stored in user's profile → served Monday

Radio / Autoplay:
  Used for continuous play after user queue empties:
  → Nearest-neighbor search in audio feature space
  → "Give me 50 songs most similar to my current session's average feature vector"
  → Uses approximate nearest neighbor (e.g., FAISS or Spotify's own Annoy library)
```

### 6. Spotify Connect (Multi-Device Session)

Unique feature: control playback on one device (phone) while audio plays on another (speaker, desktop):

```
User opens Spotify on phone while music plays on their car radio (also Spotify):
  
  All active Spotify devices registered in user's session:
  {
    "devices": [
      {"id": "car-abc", "name": "Car", "type": "AutoDevice", "is_active": true},
      {"id": "phone-xyz", "name": "My iPhone", "type": "Smartphone", "is_active": false}
    ]
  }
  
Phone can:
  - See what's playing on car
  - Pause, skip, change volume → commands sent to car device
  - Transfer playback to phone → car stops, phone starts playing at same position

Implementation:
  - Device registry in Redis (per user: list of active device IDs + WebSocket connections)
  - Control commands routed via Spotify's real-time signaling channel
  - Playback state synced via "player state" message (current track, position, volume)
```

### 7. Listening History and Play Events

```
Every song play generates a stream event:
  {user_id, track_id, timestamp, duration_ms_played, context_type, context_id, skip}

Stream events:
  → Kafka (raw event stream)
  → Flink (real-time aggregation)
    - Update real-time popularity scores
    - Detect trending tracks
    - Artist streaming dashboards (near real-time)
  → Spark batch jobs (daily)
    - Update recommendation models
    - Generate Discover Weekly playlists
    - Royalty calculations (per-stream payment to artists)
  → Data warehouse (Hive / BigQuery)
    - Analytics, A/B testing, business metrics
```

---

## Key Design Decisions

| Decision | Choice | Reasoning |
|---|---|---|
| Audio encoding | OGG Vorbis + AAC at 4 quality tiers | Device-native format; multiple quality tiers for different plans |
| Streaming protocol | Simple HTTP download (no ABR) | Small file size; buffer entire song vs. adaptive segments |
| Offline storage | Encrypted local cache with DRM + 30-day license TTL | Premium feature; prevents unauthorized redistribution |
| Recommendations | Collaborative filter + NLP + audio features | Most sophisticated in industry; key competitive differentiator |
| Multi-device (Connect) | WebSocket signaling + device registry in Redis | Real-time control commands; low-latency state sync |
| Discover Weekly generation | Batch (Saturday night), served Monday | Allows full offline batch training without real-time constraints |

---

## Scaling Challenges

| Challenge | Solution |
|---|---|
| 100M tracks × multiple bitrates served globally | CDN (Akamai) with regional PoPs; files are small — high cache efficiency |
| Discover Weekly for 250M users | Pre-generated Saturday night in batch; stored by user_id |
| 10K songs offline per device | Encrypted local cache with DRM license TTL |
| Real-time popularity updates | Kafka → Flink aggregation; write to Redis |
| Royalty calculations | Kafka stream events → Spark batch → per-track per-country accounting |

---

## Common Interview Mistakes

- **Designing ABR like YouTube**: audio files are 3–10 MB — no need for adaptive bitrate chunking. Buffer the whole song quickly, pre-fetch the next.
- **Forgetting offline mode design**: offline is a core Spotify feature (especially for mobile). Missing the encrypted local cache, DRM license TTL, and sync mechanism shows a gap.
- **No gapless playback design**: in music, gaps between tracks ruin the experience. Pre-fetching the next track at 75% of current playback eliminates gaps.
- **Only discussing collaborative filtering**: Spotify's recommendation uses three distinct signal types (collaborative filtering, NLP on playlists, audio features). Discussing all three shows depth.

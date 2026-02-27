# Design Netflix

## Problem Statement

Design a video streaming service that licenses and produces content, allows subscribers to browse a personalized catalog, and streams HD/4K video to 200 million subscribers worldwide.

**Scale requirements (Netflix)**:
- 238 million subscribers in 190 countries
- 200 million+ stream hours per day
- Peak: 15% of world's internet bandwidth during evenings
- Videos must start playing within 3 seconds
- Support 4K HDR streaming (~25 Mbps per stream)
- 99.99% availability (< 1 hour downtime per year)

---

## What's Different from YouTube

| Dimension | Netflix | YouTube |
|---|---|---|
| Content model | Licensed/original premium content | User-generated content |
| Upload volume | ~1 new title per day (processed overnight) | 500 hours/minute |
| Viewing session | Long (1–3 hour films, TV episodes) | Short to medium |
| CDN | Netflix's own CDN (Open Connect) | Google's global CDN |
| DRM | Mandatory (content protection) | Optional |
| Pre-positioning | Proactively pushed to ISP appliances | Pull-through caching |

---

## High-Level Architecture

```
Client (Smart TV, iOS, Android, Web)
   │
   ├── Discover: catalog browse, search, recommendations
   │      └──► API Servers (AWS microservices — 100+ services)
   │
   └── Stream: play a movie/show
          └──► Open Connect CDN (Netflix's own CDN appliances at ISPs)
                   │
                   └── If cache miss: origin (S3)
```

---

## How it works

### 1. Content Ingestion Pipeline

Netflix receives raw studio content (ProRes, RED, ARRIRAW — huge files):

```
Studio delivers raw master files (1 film = 2–4 TB of raw footage)
     │
     ▼
Netflix Ingest Pipeline (Apache Spark + custom encoding farm)
     │
     ├── Create 1,200+ encoding profiles per title:
     │   Video files:
     │     4K HDR (HEVC/H.265, AV1) at multiple bitrates
     │     1080p HDR, 1080p SDR
     │     720p, 480p, 360p (mobile/low-bandwidth profiles)
     │   Audio:
     │     5.1 surround, stereo, mono (Dolby Atmos where available)
     │     50+ language audio tracks
     │   Subtitles:
     │     50+ language subtitle tracks (VTT format)
     │
     ├── Per-Scene Encoding (better than fixed-bitrate):
     │   Action scene: high bitrate for motion clarity
     │   Static dialogue scene: low bitrate (visually simple)
     │   → Same quality at 30–40% lower average bitrate vs CBR
     │
     ├── DRM Encryption:
     │   → Content encrypted with Widevine (Android), FairPlay (Apple), PlayReady (Windows)
     │   → Encryption keys stored in separate secure key server (not in CDN)
     │
     └── Publish to Open Connect (CDN) appliances worldwide
```

### 2. Open Connect (Netflix's Own CDN)

Netflix runs its own CDN to control delivery quality and reduce ISP transit costs:

```
Tier 1: Open Connect Appliances (OCA) at ISPs
  - Purpose-built servers at 1,000+ ISPs worldwide
  - Storage: 100–200 TB per appliance (holds Netflix's entire active catalog)
  - Bandwidth: 100 Gbps network interfaces
  - Most ISPs host OCAs in their data centers at no charge
    (rationale: Netflix traffic = 15% of internet; owning delivery saves everyone transit costs)

Tier 2: Regional OCAs at Internet Exchange Points (IXPs)
  - For smaller ISPs without local OCAs
  - Higher latency than Tier 1 but better than origin

Tier 3: Origin (AWS S3)
  - Only used if content is not on any OCA near user (rare for popular content)
```

**Proactive Content Placement** (no pull-through caching):
```
Netflix knows what subscribers will watch tomorrow:
  → Recommendation model predicts viewing demand by title × region × day
  → Each night (off-peak hours): proactively push predicted popular titles to OCAs
  → Morning: OCAs are pre-loaded with expected popular content
  
Result: 99%+ of streams served from local OCA (no transit cost, low latency)
```

### 3. Adaptive Bitrate Streaming

Netflix uses **DASH** (Dynamic Adaptive Streaming over HTTP) + per-scene bitrate optimization:

```
manifest.mpd (DASH manifest served by API):
  - Lists all video adaptations (bitrate + resolution)
  - Lists all audio tracks (languages)
  - Lists all subtitle tracks

Client player:
  1. Request manifest from Netflix API (selects nearest OCA endpoint)
  2. Fetch first video segment (low quality for fast start)
  3. Buffer-based ABR: if buffer < 15 sec → switch down; if > 30 sec → switch up
  4. Decrypt segment using DRM license (fetched from key server once per session)
  5. Decode and render

Buffer management:
  - Target buffer: 60 seconds (longer than YouTube's 30s — Netflix optimizes for no rebuffering)
  - Aggressive pre-fetching of next episode in a TV series
```

### 4. DRM (Digital Rights Management)

Content protection is non-negotiable for Netflix's licensed content:

```
Playback session flow:

1. User clicks "Play" on movie XYZ
   │
2. Netflix API checks:
   - Valid subscription (entitlement check)
   - Correct geographic license (geo-check: content licensed per-country)
   │
3. Issue DASH manifest URL (time-limited, user-specific signed URL)
   │
4. Client fetches manifest → begins downloading encrypted segments from OCA
   │
5. Client sends DRM license request to Netflix License Server:
   - Session token, device ID, device model
   │
6. License Server returns decryption keys (AES-128)
   - Keys are valid for this session only
   - Hardware-backed key storage on device (TEE: Trusted Execution Environment)
   │
7. Player decrypts segments on-the-fly using keys stored in hardware

What OCA nodes store: encrypted video segments + encrypted audio
What OCA nodes never see: decryption keys
What Netflix servers store: decryption keys (in secure key management service)
```

### 5. Microservices Architecture

Netflix pioneered cloud-native microservices on AWS:

```
API Gateway (Zuul) → handles all client requests
     │
     ├── Account Service → subscription status, billing
     ├── Catalog Service → browse titles, metadata
     ├── Recommendation Service → machine learning, personalization
     ├── Playback Service → find best OCA, generate manifest URL
     ├── Search Service → Elasticsearch-based title search
     ├── Notification Service → email, push notifications
     └── ... (100+ services)

Key Netflix open-source infrastructure:
  Chaos Monkey / Chaos Engineering — randomly kill services in production
  Hystrix — circuit breaker (now deprecated; Resilience4j successor)
  Ribbon — client-side load balancing
  Eureka — service registry
  Zuul — API gateway
  EVCache — distributed cache built on Memcached
```

### 6. Recommendation System

Netflix's recommendation directly drives engagement (80% of viewing starts from a recommendation):

```
Signals used:
  - Viewing history (complete watches, partial watches, abandoned)
  - Rating history (explicit 👍/👎 + implicit)
  - Browsing history (search queries, title hovers)
  - Demographics (country, device, time of day)
  - Social graph (not public, but household viewing patterns)

Two-stage recommendation:
  Stage 1 (Candidate Generation):
    - Matrix factorization: user-item latent factors
    - Content-based: similar to previously watched content
    - 500–1000 candidate titles

  Stage 2 (Ranking + Personalization):
    - Contextual ranking: time of day, day of week, device type
      (action movie at 10 PM on TV; documentary on mobile at lunch)
    - Thumbnail A/B testing: different artwork per user
      (same title, different thumbnail based on predicted click-through)
    - Final ranked list of 40–60 titles for homepage rows
```

### 7. A/B Testing at Scale

Netflix runs 100s of A/B tests simultaneously:

```python
# Every user is in multiple simultaneous experiments
class ExperimentService:
    async def get_experience(self, user_id: str, feature: str) -> dict:
        # Deterministic assignment: user always gets same bucket for same experiment
        experiment = await self.get_active_experiment(feature)
        bucket_hash = hash(f"{user_id}:{experiment.id}") % 100
        
        for bucket in experiment.buckets:
            if bucket.start <= bucket_hash < bucket.end:
                return bucket.config
        
        return experiment.control_config

# Example: test new recommendation algorithm for 1% of users
# If metrics improve: gradually roll out to 5% → 20% → 100%
```

---

## Key Design Decisions

| Decision | Choice | Reasoning |
|---|---|---|
| CDN | Own CDN (Open Connect) at ISPs | Lower latency, cost savings, quality control |
| Content placement | Proactive push based on ML demand prediction | 99%+ local CDN cache hit |
| Encoding | Per-scene adaptive bitrate (not CBR) | 30–40% bandwidth savings at same quality |
| Streaming protocol | DASH + DRM (Widevine/FairPlay/PlayReady) | Industry standard; content protection required |
| Microservices | 100+ services on AWS | Independent deployment; different scaling needs |
| Recommendations | Contextual ML + thumbnail A/B testing | 80% of viewed content starts from recommendation |

---

## Scaling Challenges

| Challenge | Solution |
|---|---|
| 15% of world internet bandwidth | Own CDN (Open Connect) co-located at ISPs |
| Cold start for new region | Pre-position catalog proactively before launch |
| 99.99% availability | Chaos engineering; cross-region A/Z redundancy on AWS |
| 4K HDR streaming quality | Per-scene encoding; buffer to 60 seconds |
| Geo-restricted licensing | Entitlement check at playback session start; CDN serves by region |

---

## Common Interview Mistakes

- **Generic CDN**: saying "use CloudFront/Akamai" misses the core Netflix insight — Open Connect is co-located at ISPs for ultra-low latency and massive cost reduction.
- **Not discussing DRM**: for licensed content, this is non-negotiable. Omitting content protection shows a gap in understanding production video systems.
- **Pull-through CDN for all content**: Netflix doesn't wait for first request to cache. Proactive pre-positioning is the key efficiency insight.
- **Ignoring entitlement and geo-check**: a user in India cannot watch US-only licensed content. This check must happen at playback request time, not at catalog browse time.

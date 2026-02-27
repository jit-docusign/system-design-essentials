# Design Twitter / X Feed

## Problem Statement

Design Twitter's home timeline — the feed showing tweets from accounts a user follows, ordered reverse-chronologically (with some ranking). Users can post tweets, retweet, and reply.

**Scale requirements (Twitter at peak)**:
- 350 million registered users, 200 million DAU
- 500 million tweets posted per day (~6,000 tweets/sec)
- 28 billion timeline reads per day (~300,000 reads/sec)
- Timeline must load in < 200ms
- Write:read ratio is approximately 1:60 — reads dominate

---

## Core Features

1. **Post** a tweet (text, media, links)
2. **Follow** / unfollow users
3. **Home timeline** — tweets from followed users, reverse-chronological
4. **User timeline** — tweets by a specific user
5. **Retweet** and **like**
6. **Search** — keyword, hashtag, user search

---

## High-Level Architecture

```
Client
  │
  ▼
┌────────────────────────────────────────────────────────────┐
│                      API Gateway                           │
│  (auth, rate limiting: 300 requests/15min per user)        │
└───┬────────────────┬──────────────────┬────────────────────┘
    │                │                  │
    ▼                ▼                  ▼
Tweet Service   Timeline Service   Social Graph Service
    │                │                  │
    ▼                ▼                  ▼
Tweet Store      Timeline Cache      Follow Store
(Cassandra)      (Redis)             (Redis + MySQL)
    │
    ▼
Tweet Fanout
Service (async)
```

---

## How it works

### Data Model

```sql
-- Tweets table (write-heavy — stored in Cassandra for write scale)
CREATE TABLE tweets (
    id         BIGINT PRIMARY KEY,  -- Snowflake ID (sortable, time-embedded)
    user_id    UUID NOT NULL,
    content    TEXT,                -- max 280 chars
    media_ids  LIST<UUID>,
    reply_to   BIGINT,              -- parent tweet_id if this is a reply
    retweet_of BIGINT,              -- original tweet_id if retweet
    like_count INT,
    retweet_count INT,
    reply_count INT,
    created_at TIMESTAMP
);

-- Social graph: follows
-- Stored in both directions for fast fanout and for "is following" checks
follows_by_follower: {follower_id: [followee_id, ...]}  -- "I follow these people"
follows_by_followee: {followee_id: [follower_id, ...]}  -- "these people follow me" (for fanout)
```

**Snowflake IDs** — Twitter-invented distributed ID generation that encodes timestamp in the ID, enabling time-ordered sorting without a sort column:

```
64-bit Snowflake ID:
┌─────────────────────┬──────────────┬──────────────────────┐
│  41 bits            │  10 bits     │  12 bits             │
│  timestamp ms       │  machine ID  │  sequence number     │
│  (since epoch)      │              │  (per ms, per machine)│
└─────────────────────┴──────────────┴──────────────────────┘
- ~69 years before overflow
- Sortable by ID = sortable by time (no secondary index needed)
- 4096 IDs per millisecond per machine
```

### The Timeline Problem: Fan-out on Write vs Read

**Pure fan-out on write (pre-computed timelines)**:

```
User posts tweet
  └──► look up all followers (could be 100M for celebrities)
       └──► insert tweet_id into each follower's timeline cache

Read timeline:
  └──► Redis LRANGE timeline:{user_id} 0 799  // instant
```

**Problems with pure fan-out on write**:
- Katy Perry has 100M followers: inserting 100M Redis entries per tweet takes minutes.
- 95% of users are inactive — wasteful to fan-out to their caches.

**Pure fan-out on read (compute at read time)**:
```
Read timeline for user-456:
  1. Get list of 500 people user follows
  2. Query each person's recent tweets
  3. Merge-sort by time
  4. Return top 200

Problems: 500 DB queries per timeline read at 300K reads/sec → impossible.
```

**Twitter's actual hybrid approach**:

```python
CELEBRITY_THRESHOLD = 1_000_000  # 1M followers = celebrity

async def post_tweet(user_id: str, tweet: dict) -> str:
    tweet_id = generate_snowflake_id()
    
    # 1. Write tweet to Cassandra (the source of truth)
    await cassandra.execute(
        "INSERT INTO tweets (id, user_id, content, created_at) VALUES (?, ?, ?, ?)",
        tweet_id, user_id, tweet['content'], datetime.utcnow()
    )
    
    # 2. Fan-out to active followers (async, via Kafka)
    follower_count = await social_graph.get_follower_count(user_id)
    
    if follower_count < CELEBRITY_THRESHOLD:
        # Fan-out to all followers
        await kafka.produce('tweet-fanout', {
            'tweet_id': tweet_id,
            'author_id': user_id,
            'timestamp': tweet_id >> 22,  # extract ms from Snowflake
        })
    
    return tweet_id

# Fanout Worker (Kafka consumer)
async def fanout_worker(event: dict) -> None:
    tweet_id = event['tweet_id']
    author_id = event['author_id']
    
    # Get active followers (last login < 30 days)
    active_followers = await social_graph.get_active_followers(author_id)
    
    # Batch Redis writes
    pipeline = redis.pipeline()
    for follower_id in active_followers:
        # LPUSH and trim to 800 entries
        pipeline.lpush(f'timeline:{follower_id}', tweet_id)
        pipeline.ltrim(f'timeline:{follower_id}', 0, 799)
    await pipeline.execute()

async def get_timeline(user_id: str, cursor: int = 0, count: int = 20) -> list:
    # 1. Get pre-fanned-out timeline from Redis
    cached_tweet_ids = await redis.lrange(f'timeline:{user_id}', cursor, cursor + count - 1)
    
    # 2. Get tweets from celebrities the user follows (real-time pull)
    followed_celebrities = await social_graph.get_followed_celebrities(user_id)
    celebrity_tweets = []
    for celeb_id in followed_celebrities:
        # Pull from celeb's user timeline (also cached)
        recent = await redis.lrange(f'user_timeline:{celeb_id}', 0, 19)
        celebrity_tweets.extend(recent)
    
    # 3. Merge: union of cached feed + celebrity tweets, sorted by Snowflake ID (= time)
    all_tweet_ids = list(set(cached_tweet_ids + celebrity_tweets))
    all_tweet_ids.sort(reverse=True)  # Snowflake IDs are time-sortable
    
    # 4. Hydrate tweet content from Cassandra (batch fetch)
    tweets = await cassandra.execute(
        "SELECT * FROM tweets WHERE id IN ? LIMIT ?",
        all_tweet_ids[:count], count
    )
    
    # 5. Hydrate user data (separate users table, likely cached in Redis)
    user_ids = [t.user_id for t in tweets]
    users = await redis.mget([f'user:{uid}' for uid in user_ids])
    
    return attach_user_data(tweets, users)
```

### Timeline Cache (Redis Lists)

```
Key: timeline:{user_id}      (fan-out-populated feed)
Key: user_timeline:{user_id} (this user's own tweets, for celebrity pull)

Structure: Redis List (LPUSH for new tweets, LTRIM to 800 entries)

timeline:user-456:
  [tweet-999, tweet-998, tweet-997, ..., tweet-100]  (800 most recent)

Why List instead of Sorted Set?
  - Snowflake IDs are already time-ordered → no need for a score
  - LPUSH + LTRIM is simpler and faster than ZADD
  - LRANGE with index cursor enables efficient pagination
```

### Tweet Storage — Cassandra

Why Cassandra for tweets:
- Extremely high write throughput (500M tweets/day = 6k/sec sustained)
- Append-only workload (tweets are immutable after posting)
- Time-ordered access patterns (user timeline = recent tweets by user_id)
- No complex joins needed

```sql
-- Cassandra schema optimized for user timeline reads
CREATE TABLE tweets_by_user (
    user_id   UUID,
    tweet_id  BIGINT,   -- Snowflake: time-sortable
    content   TEXT,
    like_count INT,
    created_at TIMESTAMP,
    PRIMARY KEY (user_id, tweet_id)
) WITH CLUSTERING ORDER BY (tweet_id DESC);

-- Fetch user's recent tweets:
SELECT * FROM tweets_by_user WHERE user_id = ? LIMIT 200;
```

### Search

Full-text tweet search requires a different store from Cassandra:

```
Tweets published → Kafka → Search Indexer → Elasticsearch
                                              │
User queries "ukraine war" ─────────────────►│
                                              │
                                            BM25 full-text search
                                              + recency scoring boost
```

Twitter's search has an additional challenge: **real-time indexing**. Tweets should be searchable within seconds of posting. Elasticsearch with a Kafka-backed ingestion pipeline achieves ~1-2 second indexing latency.

---

## Key Design Decisions

| Decision | Choice | Reasoning |
|---|---|---|
| Tweet IDs | Snowflake (time-ordered) | Enable time-sorted scans without timestamp index |
| Tweet storage | Cassandra | High write throughput, append-optimized workload |
| Timeline | Redis List (fan-out on write) | Read:write is 60:1 — optimize for reads |
| Celebrity problem | Hybrid (pull celebrities at read time) | Avoids O(followers) writes for mega-accounts |
| Active follower filter | Skip inactive users (>30 days) in fan-out | Reduces Redis memory and fan-out compute by ~70% |

---

## Scaling Challenges

| Challenge | Solution |
|---|---|
| 300K timeline reads/sec | Redis cache; batch tweet hydration |
| 100M-follower celebrity posts | Pull at read time; no fan-out |
| 500M tweets/day write throughput | Cassandra (distributed, write-optimized) |
| Real-time search | Kafka → Elasticsearch pipeline; ~1s latency |
| Social graph traversal | Follow graph in Redis + MySQL; cached by user_id |

---

## Hard-Learned Engineering Lessons

- **Not mentioning Snowflake IDs**: explaining how you sort tweets without a timestamp index is a differentiator. Snowflake IDs enable this naturally.
- **Fan-out to inactive users**: naively fanning out to all followers wastes memory. Mention active-user filtering.
- **Forgetting tweet hydration**: returning tweet_ids from the timeline cache is only the first step. You still need to fetch tweet content and user metadata — batch these into a single I/O round-trip.
- **Same model for timeline and search**: Cassandra cannot do full-text search. Archive tweets to Elasticsearch separately for search use cases.

# Design a Gaming Leaderboard

## Problem Statement

Design a real-time leaderboard that shows players their global rank and nearby competitors in a game with 100 million players. Players want to see their rank update immediately after scoring, and browse the top-100 players globally.

**Scale requirements**:
- 100 million players globally
- 1 million score updates per minute (~16,700/sec)
- Query top-100 global leaderboard: < 10ms
- Query a player's rank: < 10ms
- Query players ranked N-5 to N+5 around a specific player: < 10ms
- Support multiple leaderboards: global, daily, weekly, friends

---

## Why This Is Non-Trivial

With 100 million players, a naive approach (sort by score in a DB) hits problems:

```sql
-- WRONG: Full table scan on every rank query
SELECT COUNT(*) + 1 AS rank
FROM scores
WHERE score > (SELECT score FROM scores WHERE player_id = 'player_42');

-- At 100M players: 100M row scan for every rank query. Seconds of latency.
-- 16,700 score updates/sec + 100M rank queries/sec = DB melts
```

The right solution: **Redis Sorted Set** (ZSET) — designed exactly for this.

---

## Redis Sorted Set (Core Data Structure)

```
Redis Sorted Set internals:
  - Hash map: member → score (O(1) score lookup)
  - Skip list: ordered by score (O(log N) insert, rank query)
  
Key operations:
  ZADD leaderboard 9500 "player_42"     → Insert/update player's score: O(log N)
  ZSCORE leaderboard "player_42"         → Get player's score: O(1)
  ZRANK leaderboard "player_42"          → Player's rank (0-indexed, ascending): O(log N)
  ZREVRANK leaderboard "player_42"       → Rank (0-indexed, descending = 1st, 2nd): O(log N)
  ZREVRANGE leaderboard 0 99             → Top 100 players: O(log N + 100)
  ZREVRANGEBYSCORE leaderboard +inf -inf LIMIT 0 100 → Same with score filter
  ZREVRANGE leaderboard N-5 N+5         → Players around rank N: O(log N + 11)
  
For 100M players:
  ZADD: O(log 100M) ≈ 27 comparisons → microseconds
  ZREVRANK: O(log 100M) ≈ 27 comparisons → microseconds
```

```python
import redis
import time

r = redis.Redis(host="redis-leaderboard", port=6379, decode_responses=True)

GLOBAL_LB = "lb:global"
DAILY_LB = lambda: f"lb:daily:{time.strftime('%Y-%m-%d')}"
WEEKLY_LB = lambda: f"lb:weekly:{time.strftime('%Y-W%W')}"

def update_score(player_id: str, score: int, delta: bool = False):
    """
    Update a player's score.
    If delta=True, add to existing score (e.g., +500 points from this match).
    If delta=False, set absolute score.
    """
    pipe = r.pipeline()
    
    if delta:
        pipe.zincrby(GLOBAL_LB, score, player_id)
        pipe.zincrby(DAILY_LB(), score, player_id)
        pipe.zincrby(WEEKLY_LB(), score, player_id)
    else:
        pipe.zadd(GLOBAL_LB, {player_id: score})
        pipe.zadd(DAILY_LB(), {player_id: score})
        pipe.zadd(WEEKLY_LB(), {player_id: score})
    
    pipe.execute()  # All three updates atomically in one round-trip

def get_rank(player_id: str) -> dict:
    """Get player's rank (1-indexed) and score"""
    pipe = r.pipeline()
    pipe.zrevrank(GLOBAL_LB, player_id)  # 0-indexed rank by score descending
    pipe.zscore(GLOBAL_LB, player_id)
    results = pipe.execute()
    
    rank_0indexed, score = results
    if rank_0indexed is None:
        return None  # Player has no score yet
    
    return {
        "rank": rank_0indexed + 1,     # Convert to 1-indexed
        "score": int(score),
        "player_id": player_id
    }

def get_top_k(k: int = 100) -> list[dict]:
    """Get top K players with ranks"""
    # ZREVRANGE returns [(player_id, score), ...]
    entries = r.zrevrange(GLOBAL_LB, 0, k - 1, withscores=True)
    
    return [
        {"rank": i + 1, "player_id": pid, "score": int(score)}
        for i, (pid, score) in enumerate(entries)
    ]

def get_players_around(player_id: str, window: int = 5) -> list[dict]:
    """Get players ranked within ±window positions of player"""
    rank_0indexed = r.zrevrank(GLOBAL_LB, player_id)
    if rank_0indexed is None:
        return []
    
    start = max(0, rank_0indexed - window)
    end = rank_0indexed + window
    
    entries = r.zrevrange(GLOBAL_LB, start, end, withscores=True)
    
    return [
        {"rank": start + i + 1, "player_id": pid, "score": int(score),
         "is_current_player": pid == player_id}
        for i, (pid, score) in enumerate(entries)
    ]
```

---

## Memory Estimation

```
100 million players in Redis Sorted Set:

Each entry: ~60 bytes (8-byte score + ~20-byte player_id string + skip list pointers)
100M × 60 bytes = 6 GB per leaderboard

With global + daily + weekly = ~18 GB 
(comfortably fits in a single Redis node; replica for HA adds 2×)

Redis RAM: 32 GB instance handles this with room to spare.
```

---

## Sharding for Multi-Leaderboard Scale

When you have thousands of leaderboards (per-game, per-season, per-tournament):

```python
class ShardedLeaderboard:
    """
    Shard leaderboards across multiple Redis nodes.
    Each leaderboard lives entirely on one shard (no cross-shard ZADD needed).
    """
    
    def __init__(self, redis_nodes: list[redis.Redis], shards: int = 16):
        self.nodes = redis_nodes
        self.shards = shards
    
    def get_shard(self, leaderboard_id: str) -> redis.Redis:
        shard_idx = hash(leaderboard_id) % self.shards
        return self.nodes[shard_idx % len(self.nodes)]
    
    def update_score(self, leaderboard_id: str, player_id: str, score: int):
        node = self.get_shard(leaderboard_id)
        node.zadd(f"lb:{leaderboard_id}", {player_id: score})
    
    def get_rank(self, leaderboard_id: str, player_id: str) -> int:
        node = self.get_shard(leaderboard_id)
        rank = node.zrevrank(f"lb:{leaderboard_id}", player_id)
        return (rank or 0) + 1
```

---

## Friends Leaderboard

Show a player's rank among their friends (not global rank):

```python
def get_friends_leaderboard(player_id: str, friend_ids: list[str]) -> list[dict]:
    """
    Get scores for all friends and rank among them.
    
    Approach 1: Fetch all friend scores + sort in application
    → Works for up to ~10,000 friends
    
    Approach 2: Redis ZINTERSTORE (intersection of player's scores set)
    → More complex but fully in Redis
    """
    # Approach 1: Fetch and sort (simpler)
    all_players = friend_ids + [player_id]
    scores = r.zmscore(GLOBAL_LB, all_players)  # O(N) for N friends
    
    player_scores = []
    for pid, score in zip(all_players, scores):
        if score is not None:
            player_scores.append((pid, int(score)))
    
    # Sort by score descending
    player_scores.sort(key=lambda x: -x[1])
    
    return [
        {
            "rank": i + 1,
            "player_id": pid,
            "score": score,
            "is_current_player": pid == player_id
        }
        for i, (pid, score) in enumerate(player_scores)
    ]
```

---

## Tie-Breaking

Two players with the same score must have a consistent rank. Options:

```python
# Option 1: Timestamp tiebreak (earlier achiever ranks higher)
def update_score_with_tiebreak(player_id: str, score: int):
    """
    Encode score + timestamp into a single float.
    Higher score ranks higher; among equals, earlier timestamp wins.
    
    Technique: use score as integer part, timestamp embedded in fractional
    score_with_tiebreak = score + (MAX_TS - timestamp) / MAX_TS_SCALE
    """
    MAX_TS = 10**10   # Large enough to not overflow float precision
    ts = int(time.time())
    tiebreak = (MAX_TS - ts) / MAX_TS  # Earlier time → larger fraction → higher rank
    
    composite_score = score + tiebreak
    r.zadd(GLOBAL_LB, {player_id: composite_score})
    # Recoverable: score = int(composite_score)

# Option 2: Explicit rank assignment (Lua script for atomicity)
RANK_WITH_TIE_SCRIPT = """
local score = tonumber(ARGV[1])
local player_id = ARGV[2]
local current = redis.call('ZSCORE', KEYS[1], player_id)
if current and tonumber(current) >= score then
    return false  -- don't downgrade score
end
redis.call('ZADD', KEYS[1], score, player_id)
return true
"""
```

---

## Daily / Weekly Leaderboard Reset

```python
import schedule

def rotate_daily_leaderboard():
    """
    At midnight UTC: archive yesterday's leaderboard, create fresh one.
    Yesterday's leaderboard key expires after 7 days.
    """
    yesterday_key = f"lb:daily:{(datetime.utcnow() - timedelta(days=1)).strftime('%Y-%m-%d')}"
    
    # Set expiry on yesterday's leaderboard (keep for 7 days for stats)
    redis.expire(yesterday_key, 7 * 86400)
    
    # Announce winners from yesterday
    top_10 = redis.zrevrange(yesterday_key, 0, 9, withscores=True)
    announce_winners(top_10)
    
    # New daily leaderboard starts fresh (empty ZADD will auto-create it)
    print(f"Daily leaderboard rotated. New key: lb:daily:{datetime.utcnow().strftime('%Y-%m-%d')}")

schedule.every().day.at("00:00").do(rotate_daily_leaderboard)
```

---

## Persistence and Durability

Redis is in-memory. For financial/competitive leaderboards where data loss is unacceptable:

```
Redis persistence options:
  1. RDB snapshots (periodic): save to disk every 60 seconds
     Risk: up to 60 seconds of score updates lost on crash
     
  2. AOF (Append-Only File): log every write command to disk
     Risk: near-zero data loss (last write might be lost); higher disk I/O
     
  3. Redis Sentinel / Cluster with replicas:
     Primary fails → replica promoted with all data
     AOF on replicas for additional durability
     
For gaming leaderboards:
  - Losing 60 seconds of scores in a crash is usually acceptable
  - Use RDB + replicas (1 primary, 2 replicas)
  - Write important score updates (tournament scores) to PostgreSQL as source of truth
  - Rebuild Redis from PostgreSQL if corrupted
```

---

## Key Design Decisions

| Requirement | Solution |
|---|---|
| O(log N) rank query for 100M players | Redis Sorted Set (skip list) |
| Score update throughput (16K/sec) | Redis pipelined ZINCRBY |
| Friends leaderboard | Fetch friend scores with ZMSCORE + app-side sort |
| Multiple time windows | Separate ZSET per time window (daily, weekly) |
| Durability | Redis AOF + replica + PostgreSQL score backup |
| Tie-breaking | Composite float score (score + tiebreak fraction) |

---

## Common Interview Mistakes

1. **Using a relational DB with ORDER BY for rank**: `SELECT rank FROM (SELECT player_id, RANK() OVER (ORDER BY score DESC) AS rank FROM scores) WHERE player_id = 'X'` — this scans 100M rows on every request.

2. **Not knowing Redis ZREVRANK**: many candidates know ZADD but forget ZREVRANK for O(log N) rank lookups.

3. **Forgetting memory estimation**: 100M entries in Redis is ~6 GB — fits in memory. Show you did the math.

4. **Not distinguishing global vs. friends vs. seasonal leaderboards**: these have different architectures. Friends leaderboard can't be a global ZSET; it's a per-query computation.

5. **Ignoring tie-breaking**: two players with the same score need a consistent, fair tiebreak (first achiever, player ID lexicographic, etc.).

6. **Not discussing leaderboard rotation**: daily/weekly leaderboards must be reset. Discuss key rotation strategy and archival of past leaderboards.

7. **Using integer scores without considering float precision**: Redis scores are 64-bit floats. Storing tiebreak information in the fractional part requires understanding float64 precision limits (~15-16 significant decimal digits).

# Design a Typeahead / Autocomplete System

## Problem Statement

Design the autocomplete suggestion service behind a search bar. As a user types "iph", the system should instantly return ["iPhone 15", "iPhone case", "iPhone charger", ...] ranked by popularity. Used in Google Search, Amazon product search, Twitter search, and IDEs like VS Code.

**Scale requirements**:
- 100 million active users
- 50,000 search queries per second (peak)
- p99 suggestion latency < 50ms
- Support 100 million distinct query strings
- Update suggestion rankings within 10 minutes of trending queries

---

## What It Is

A typeahead system matches user keystrokes against a prefix index of known query strings ranked by popularity. It is distinct from full-text search (which finds documents containing a term) — here we complete the query itself before the user has finished typing.

---

## Trie Data Structure (Foundation)

### Conceptual Trie

```
Stored queries: "apple", "apple watch", "apple tv", "application", "app store"

         root
          │
          a
          │
          p
          │
          p ──────────── (app)
         / \
        l   s
        │   │
        e   t → o → r → e  [app store: score=5000]
        │
       [apple: score=9000]
        |
        ├── w → a → t → c → h  [apple watch: score=7000]
        └── t → v              [apple tv: score=3000]

        a → p → p → l → i → c → a → t → i → o → n  [application: score=2000]

Query "app" → traverse to node "app" → DFS all descendants → return top-K by score:
  1. apple (9000)
  2. apple watch (7000)
  3. app store (5000)
  4. apple tv (3000)
  5. application (2000)
```

### Optimized Trie: Store Top-K at every node

Instead of DFS on every query, cache top-K suggestions at every prefix node:

```python
class TrieNode:
    def __init__(self):
        self.children: dict[str, TrieNode] = {}
        self.is_end: bool = False
        self.top_k: list[tuple[int, str]] = []  # top-5 (score, query string)

class Trie:
    def __init__(self, k=5):
        self.root = TrieNode()
        self.k = k
    
    def insert(self, query: str, score: int):
        node = self.root
        for char in query:
            if char not in node.children:
                node.children[char] = TrieNode()
            node = node.children[char]
            # Update top-K at every node along path
            self._update_topk(node, score, query)
        node.is_end = True
    
    def _update_topk(self, node: TrieNode, score: int, query: str):
        # Maintain sorted list of top-K by score
        entries = node.top_k
        entries.append((score, query))
        entries.sort(key=lambda x: -x[0])
        node.top_k = entries[:self.k]
    
    def search(self, prefix: str) -> list[str]:
        node = self.root
        for char in prefix:
            if char not in node.children:
                return []           # no matching prefix
            node = node.children[char]
        return [query for _, query in node.top_k]  # O(1) lookup!

# Usage
trie = Trie(k=5)
trie.insert("apple", 9000)
trie.insert("apple watch", 7000)
trie.insert("app store", 5000)
trie.insert("apple tv", 3000)
trie.insert("application", 2000)

trie.search("app")  
# → ["apple", "apple watch", "app store", "apple tv", "application"]
```

**Insert** is O(query_length × k log k) — done offline in batch  
**Search** is O(prefix_length) — blazing fast for real-time queries

---

## System Architecture

```
Client (browser/app)
       │
       │  HTTP/WebSocket (per-keystroke with debounce)
       ▼
  ┌──────────────────┐
  │  API Gateway /   │
  │  Load Balancer   │
  └────────┬─────────┘
           │
     ┌─────┴──────┐
     │            │
     ▼            ▼
  Suggestion   Suggestion     (Suggestion Service: stateless, reads from cache)
  Service 1    Service 2
     │            │
     └─────┬──────┘
           │
           ▼
  ┌──────────────────────────────────────────────────────┐
  │  Redis Cluster (prefix cache + ZRANGEBYLEX)           │
  │                                                      │
  │  Global suggestions: ZSET "suggest:global"            │
  │  Member: "apple|iphone 15"  Score: 9800               │
  │  Member: "apple|apple watch" Score: 7200              │
  │                                                      │
  │  Per-user suggestions: ZSET "suggest:user:u42"        │
  └───────────────────────────┬──────────────────────────┘
                              │  (cache miss)
                              ▼
  ┌──────────────────────────────────────────────────────┐
  │  Trie Service (in-memory Trie, sharded by prefix)    │
  │  Rebuilt every 10 minutes from batch scoring job     │
  └───────────────────────────┬──────────────────────────┘

  ┌──────────────────────────────────────────────────────┐
  │  Query Log Pipeline (async, background)              │
  │  User searches → Kafka → Flink aggregation →         │
  │  query frequency scores → S3 (hourly snapshots)      │
  │  → Trie rebuild job (every 10 min)                   │
  └──────────────────────────────────────────────────────┘
```

---

## How It Works

### 1. Client-Side Debouncing

Don't send a request on every keystroke. Wait until typing pauses:

```javascript
// Debounce: fire only after 200ms pause in typing
let timer;
const debounce = (fn, delay) => {
    clearTimeout(timer);
    timer = setTimeout(fn, delay);
};

searchInput.addEventListener('input', () => {
    debounce(() => {
        const prefix = searchInput.value.trim().toLowerCase();
        if (prefix.length >= 2) {
            fetchSuggestions(prefix);
        }
    }, 200);  // 200ms debounce
});
```

Also:
- Cancel in-flight requests if a newer one is made (AbortController)
- Cache recent prefix responses client-side (last 20 prefixes in a Map)

### 2. Redis ZRANGEBYLEX for O(log N) Prefix Search

An alternative to a Trie is storing all query strings in a Redis Sorted Set with score=0 and using lexicographic range search:

```python
import redis

r = redis.Redis()

# Index time: add all query strings with score=0 (lex order only matters)
queries = ["apple", "apple watch", "app store", "apple tv", "application"]
pipe = r.pipeline()
for q in queries:
    pipe.zadd("autocomplete", {q: 0})
pipe.execute()

# Search time: find all completions of prefix "app"
# ZRANGEBYLEX key "[app" "[app\xff" (matches any string starting with "app")
def get_suggestions_lex(prefix: str, limit: int = 10) -> list[str]:
    lower = f"[{prefix}"
    upper = f"[{prefix}\xff"  # \xff = highest Unicode code point suffix
    results = r.zrangebylex("autocomplete", lower, upper, start=0, num=limit)
    return [r.decode() for r in results]

get_suggestions_lex("app")
# → ["app store", "apple", "apple tv", "apple watch", "application"]
```

**Limitation**: This returns results in lexicographic order, not by popularity. To rank by score, you combine this with a ranking step:

```python
def get_ranked_suggestions(prefix: str, k=5):
    # 1. Get top-100 lexicographic matches
    candidates = get_suggestions_lex(prefix, limit=100)
    
    # 2. Look up scores from a hash (query → score)
    scores = {}
    pipe = r.pipeline()
    for c in candidates:
        pipe.hget("query_scores", c)
    results = pipe.execute()
    
    for query, score in zip(candidates, results):
        scores[query] = int(score or 0)
    
    # 3. Return top-K by score
    return sorted(candidates, key=lambda q: -scores.get(q, 0))[:k]
```

### 3. Trie Sharding for Scale

A single Trie with 100 million queries doesn't fit in one machine's memory. Shard by prefix:

```
Shard assignment by first character of prefix:
  Shard 1: a-e   (queries starting with a, b, c, d, e)
  Shard 2: f-k
  Shard 3: l-p
  Shard 4: q-z

For prefix "iph":
  → Route to Shard 2 (i is in f-k range)
  → Ask Shard 2's in-memory Trie for top-K("iph")

For single-character prefixes or short prefixes (highly popular):
  → Cache aggressively in Redis or CDN
  → "a" returns the same top-10 for all users → cache globally
```

### 4. Scoring Queries

The ranking signal is **search frequency × recency decay**:

```python
from datetime import datetime, timedelta
import math

def weighted_score(query_stats: dict) -> float:
    """
    query_stats = {
        "count_7d": 50000,    # searches in last 7 days
        "count_30d": 180000,  # searches in last 30 days
        "count_1y": 1200000,  # searches in last year
        "last_searched": "2024-01-15T10:00:00Z"
    }
    """
    # Exponential decay: recent searches weighted more
    w7  = 1.0   # last 7 days: full weight
    w30 = 0.5   # 7-30 days: half weight
    w1y = 0.1   # 30d-1y: 10% weight
    
    score = (query_stats["count_7d"] * w7 +
             (query_stats["count_30d"] - query_stats["count_7d"]) * w30 +
             (query_stats["count_1y"] - query_stats["count_30d"]) * w1y)
    
    return score

# Trending boost: if query grew 10x in 24h, boost it
def trending_score(query: str, hourly_counts: list[int]) -> float:
    if len(hourly_counts) < 2:
        return 1.0
    recent = sum(hourly_counts[-6:])   # last 6 hours
    baseline = sum(hourly_counts[-30:-6]) / 4  # prior 24h normalized
    if baseline == 0:
        return 2.0  # brand new trending query
    growth = recent / baseline
    trending_boost = math.log(1 + growth)  # log scale to avoid extreme boosts
    return 1.0 + trending_boost
```

### 5. Trie Rebuild Pipeline

```
Every 10 minutes:

1. Flink streaming job:
   - Consumes from Kafka "user.searches" topic
   - Aggregates: ("query", 10-min window) → count
   - Writes to "query_counts" Redis Hash: HINCRBY query_counts "iphone 15" 450

2. Batch scoring job (Spark, runs hourly):
   - Read Redis query_counts + historical counts from S3
   - Compute weighted_score for all queries
   - Write "query_scores" sorted list to S3: s3://autocomplete/scores/latest.json

3. Trie builder (every 10 min):
   - Load latest scores from S3
   - Build in-memory Trie with top-K cached at each node
   - Hot-swap: new Trie replaces old atomically (pointer swap)
   - All Trie service instances reload via config push

Note: Trie rebuild is NOT on the hot path — suggestions are served from
      the existing Trie while rebuild happens in background.
```

### 6. Personalization

Layer personalized suggestions on top of global suggestions:

```
Final suggestions = blend(personal, global):

User "u42" has searched: "python tutorial", "python list comprehension", "django"

When typing "py":
  Global top-5: ["python", "python tutorial", "python 3", "python snake", "pypi"]
  Personal top-5: ["python list comprehension", "python tutorial", "pypi"]

  Merge: deduplicate, rank personal results higher:
  → ["python list comprehension" (personal boost +2),
     "python tutorial" (both +1.5),
     "python" (global only),
     "python 3" (global only),
     "pypi" (both)]

Personal model:
  - Stored in Redis as ZSET "suggest:user:u42" (10 entries max)
  - Updated on every completed search
  - TTL 30 days (stale preferences expire)
  - Only used if prefix is >= 3 chars (short prefixes → global only)
```

---

## Key Design Decisions

| Decision | Option A | Option B | Choice |
|---|---|---|---|
| Prefix storage | Trie (in-memory) | Redis ZRANGEBYLEX | Trie for large scale; Redis for simplicity |
| Ranking granularity | Rebuild every 10 min | Real-time streaming update | 10 min rebuild (real-time has higher complexity) |
| Personalization | Per-keystroke personalization | Blend global + cached personal | Blend (personalization is a light overlay) |
| Shard strategy | Shard by first 1-2 chars | Consistent hash of prefix | First chars (locality; simple routing) |
| Client debounce | 100ms | 200ms | 200ms (balance responsiveness vs server load) |

---

## Scaling Challenges

| Challenge | Problem | Solution |
|---|---|---|
| Short-prefix hot spots | "a" prefix: hit by millions of users | Cache in Redis/CDN; single-char prefixes returned from edge |
| Trie memory per shard | 100M queries, avg 15 chars = 1.5 GB of strings alone | Compress Trie; only index top-5M queries per language |
| Multi-language support | Each language needs its own Trie | Separate Tries per locale; route by Accept-Language header |
| Trending queries | "earthquake SF" needs to appear within 2 min | Real-time path: Kafka → Flink → update Redis ZSET directly (bypass Trie for top trending) |
| Profanity / policy | Users type offensive queries | Filter at indexing time: blocklist applied before Trie insert |

---

## Hard-Learned Engineering Lessons

1. **Not debouncing the client**: saying "send a request on every keystroke" means ~10 RPS per user × 50K active users = 500K RPS for just one fast typist. Debounce brings this to ~1 RPS/user.

2. **Forgetting the "Top-K at every node" optimization**: describing a Trie where every search requires DFS is O(number of completions) which can be millions. Pre-caching top-K makes every lookup O(prefix_length).

3. **Assuming a single machine**: a Trie for all English queries comfortably fits in a few GB, but for global multi-language at 100M queries it doesn't. Show you know to shard.

4. **Ignoring ranking**: returning suggestions in lexicographic order (alphabetical) rather than by popularity is a fundamental UX failure — users always expect the most relevant result first, not the alphabetically earliest.

5. **Conflating typeahead with search**: typeahead completes the query string; full-text search finds documents. They often coexist but have very different architectures.

6. **Not discussing updates**: a Trie is efficient for read but expensive to update. Explain the offline rebuild approach to avoid complex online Trie updates under write load.

7. **Forgetting personalization tradeoffs**: personalized typeahead requires user-specific state — can't serve from a shared CDN/cache. Discuss the boundary between global cached results and personalized results.

# Design a Web Search Engine

## Problem Statement

Design a web search engine that crawls the internet, indexes pages, and returns relevant results for user queries in milliseconds.

**Scale requirements (Google-scale)**:
- 30 trillion web pages indexed
- 8.5 billion searches per day (~100,000 queries/sec)
- Search results in < 200ms
- Index freshes for news within minutes, general web within days/weeks
- 15-second average crawl-to-index latency for major news sites

---

## Core Components

1. **Web Crawler** — discover and download web pages
2. **Document Store** — store raw HTML pages
3. **Indexer** — extract text, build inverted index
4. **Ranking** — score results by relevance (PageRank + hundreds of signals)
5. **Query Serving** — accept query, retrieve results, rank, return in < 200ms

---

## How it works

### 1. Web Crawler

```
Seed URLs
    │
    ▼
┌─────────────────────────────────────────────────────┐
│                   Crawler                           │
│                                                     │
│  URL Frontier (priority queue)                      │
│  ┌─────────────────────────────────────────────┐   │
│  │  Priority 1: High-authority sites (NYT,     │   │
│  │              Wikipedia) - crawl every hour  │   │
│  │  Priority 2: Regular sites - daily/weekly   │   │
│  │  Priority 3: New/unknown sites - once/month │   │
│  └─────────────────────────────────────────────┘   │
│         │                                           │
│         ▼                                           │
│  Fetch Worker Pool (thousands of threads)           │
│    - Respect robots.txt                             │
│    - Rate limit per domain (1 req/sec default)      │
│    - Timeout: 10 seconds                            │
│    - Follow redirects (max 5)                       │
│         │                                           │
│         ▼                                           │
│  Extract links from HTML → add to Frontier         │
│  Store HTML in Document Store                      │
└─────────────────────────────────────────────────────┘
```

```python
import asyncio
import aiohttp
from url_normalize import url_normalize
from bloom_filter import BloomFilter

class Crawler:
    def __init__(self):
        self.frontier = PriorityQueue()  # (priority, url)
        self.seen_urls = BloomFilter(capacity=1_000_000_000, error_rate=0.01)
        self.doc_store = DocumentStore()  # S3 + metadata in MySQL
    
    async def crawl(self):
        async with aiohttp.ClientSession() as session:
            while True:
                priority, url = await self.frontier.get()
                
                if self.seen_urls.check(url):
                    continue  # Already crawled
                
                # Respect robots.txt
                if not await self.is_allowed(url):
                    continue
                
                # Rate limit per domain
                await self.rate_limiter.wait(get_domain(url))
                
                try:
                    html = await self.fetch(session, url, timeout=10)
                    content_hash = hash_content(html)
                    
                    # Dedup: skip if content unchanged since last crawl
                    if not await self.doc_store.is_changed(url, content_hash):
                        continue
                    
                    # Store HTML
                    doc_id = await self.doc_store.save(url, html, content_hash)
                    
                    # Mark seen
                    self.seen_urls.add(url)
                    
                    # Extract and enqueue new links
                    links = extract_links(html, base_url=url)
                    for link in links:
                        normalized = url_normalize(link)
                        priority = compute_priority(normalized)  # baseed on PageRank estimate
                        await self.frontier.put((priority, normalized))
                    
                except (aiohttp.ClientError, asyncio.TimeoutError):
                    await self.schedule_retry(url, delay_hours=24)
```

**Crawler challenges**:
- **Politeness**: don't overwhelm any single server. Respect robots.txt. Max 1 req/sec per domain.
- **URL deduplication**: Bloom filter for fast "seen it?" check at scale (1 trillion URLs).
- **Content deduplication**: same article syndicated to 1000 sites. Use SimHash to detect near-duplicates.
- **Spider trap**: sites with infinite URLs (e.g., calendar pages generating infinite date URLs). Detect and limit depth.

### 2. Document Store

```
Raw HTML stored in S3 (archive + reprocess on indexer changes)
Metadata in MySQL:
  {url, doc_id, crawl_timestamp, content_hash, content_length, http_status, last_modified}

Document processing pipeline (async, after storage):
  HTML → Parse DOM → Extract:
    - title, meta description, h1/h2 headers
    - Main body text (remove boilerplate: navigation, footer, ads)
    - Outbound links (for PageRank graph)
    - Structured data (schema.org JSON-LD)
    - Language detection
    - Canonical URL (dedup)
```

### 3. Inverted Index

The inverted index maps every word to the list of documents it appears in:

```
Forward index (doc → words):
  doc-123: ["apple", "announces", "new", "iphone", "model"]
  doc-456: ["iphone", "review", "best", "smartphone"]

Inverted index (word → docs):
  "iphone" → [doc-123, doc-456, doc-789, ...]
  "apple"  → [doc-123, doc-1200, doc-5441, ...]

Posting list entry (per document in index):
  {doc_id, term_frequency, field (title/body/url), positions}
  
  Positions: for phrase matching ("new york" must be adjacent)
  Field: title match weighted higher than body match
```

**Building the index at web scale**:
```
Step 1: MapReduce (or Spark) over all crawled documents:
  Map: emit (term, doc_id, TF, field, positions)
  Reduce: collect postings per term → sorted posting list

Step 2: Index sharding
  Shard by term hash: all postings for "iphone" go to shard 5
  Replicate each shard for redundancy

Step 3: Index serving
  Query hits all term shards in parallel → merge results
```

### 4. Ranking — The Core Differentiator

TF-IDF tells you what documents contain your query terms. Ranking tells you which are most relevant.

**TF-IDF** (baseline):
$$\text{TF-IDF}(t, d) = \text{TF}(t, d) \times \log\left(\frac{N}{\text{DF}(t)}\right)$$

Where:
- $\text{TF}(t, d)$ = term frequency in document
- $N$ = total documents
- $\text{DF}(t)$ = number of documents containing term $t$

**PageRank** (link authority):
$$\text{PR}(A) = (1 - d) + d \sum_{i} \frac{\text{PR}(T_i)}{C(T_i)}$$

Where $d$ = damping factor (0.85), $T_i$ = pages linking to $A$, $C(T_i)$ = outbound links from $T_i$.

PageRank captures: a page linked to by many high-authority pages is itself authoritative.

**Google's 200+ ranking signals** include:
- TF-IDF score per term
- PageRank (link authority)
- Anchor text (what words people use to link to this page)
- Query-document relevance (BERT/neural embeddings since 2019)
- Freshness (how recently was the page updated)
- Page experience: Core Web Vitals (load speed, layout stability)
- HTTPS (minor signal since 2014)
- User signals: click-through rate, dwell time (via Search Quality Feedback)

### 5. Query Serving (< 200ms SLA)

```
User queries "best phone 2024"

Query Processing:
  1. Spell correction: "phoen" → "phone"
  2. Query understanding: entity linking, intent detection
     "best phone" → intent: product recommendation
  3. Query expansion: "phone" also matches "smartphone", "mobile device"

Index Lookup (parallel):
  Query terms: ["best", "phone", "2024"]
  Shard lookup (in parallel for all terms):
    shard for "best"  → [{doc-123, TF=2, title}, {doc-456, TF=1, body}, ...]
    shard for "phone" → [{doc-456, TF=5, title}, {doc-789, TF=3, body}, ...]
    shard for "2024"  → [{doc-456, TF=1, title}, ...]

Merge:
  DAAT (Document-at-a-time): process posting lists in parallel
  Score each doc: TF-IDF + PageRank + other signals
  Candidate set: top-1000 by initial score

Ranking:
  Apply full ranking model (LambdaMART or neural) on top-1000 candidates
  Personalization: user's search history, location
  Final top-10 results

Snippet Generation:
  For each result: extract relevant excerpt (passage retrieval)
  Highlight query terms in snippet
```

---

## Key Design Decisions

| Decision | Choice | Reasoning |
|---|---|---|
| URL dedup | Bloom filter (1B URLs) | Sub-millisecond lookup; small memory; false positives acceptable |
| Content dedup | SimHash (near-duplicate detection) | Exact hash misses rephrased duplicates |
| Index storage | MapReduce → distributed posting lists | Only way to build at 30T page scale |
| Query SLA | Index sharding → parallel lookup | All term shards hit simultaneously → merge |
| Ranking | Multi-signal: TF-IDF + PageRank + neural | Any single signal is gameable; ensemble is robust |

---

## Hard-Learned Engineering Lessons

- **Not designing crawl politeness**: a crawler that doesn't respect robots.txt and rate limits will get blocked. It's a required component.
- **Single-machine index**: discussing how to fit 30 trillion pages in memory of one server doesn't work. Sharding is mandatory.
- **Only TF-IDF for ranking**: TF-IDF alone was solved in 1972. Mention PageRank (link structure), neural signals (BERT), and user feedback signals to show modern search knowledge.
- **Forgetting near-duplicate detection**: the same article appears on thousands of news sites. Without SimHash deduplication, search results would be full of copies.

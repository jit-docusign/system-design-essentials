# Design a Web Crawler

## Problem Statement

Design a distributed web crawler that systematically downloads and indexes content from the entire World Wide Web. This is the foundation of search engines, link checkers, archival tools (Wayback Machine), and SEO analysis tools.

**Scale requirements**:
- Crawl 1 billion web pages per day (~11,600 pages/second)
- Re-crawl each page every 30 days on average
- Crawling budget: 5 billion pages total in the index
- Be polite: respect robots.txt, don't overload any single domain
- Near-duplicate detection: don't store similar pages twice

---

## Architecture Overview

```
                        ┌─────────────────────────────┐
                        │      Seed URL Queue          │
                        │  (initial list of domains)   │
                        └──────────────┬──────────────┘
                                       │
                        ┌──────────────▼──────────────┐
                        │      URL Frontier            │
                        │  Priority queue per domain   │
                        │  (rate limiting: 1 req/sec   │
                        │   per domain)                │
                        └──────────────┬──────────────┘
                                       │
                          ┌────────────┴───────────┐
                          │                        │
               ┌──────────▼─────────┐   ┌─────────▼──────────┐
               │   Fetcher Pool     │   │   Fetcher Pool      │
               │  (1,000 threads)   │   │  (1,000 threads)    │
               └──────────┬─────────┘   └─────────┬──────────┘
                          │                        │
                          └────────────┬───────────┘
                                       │
                        ┌──────────────▼──────────────┐
                        │   Raw HTML Storage           │
                        │   (S3 / HDFS)                │
                        └──────────────┬──────────────┘
                                       │
                    ┌──────────────────┼────────────────────┐
                    │                  │                    │
         ┌──────────▼──────┐  ┌────────▼───────┐  ┌───────▼──────┐
         │  HTML Parser    │  │  Link Extractor │  │ Dedup Filter │
         │  (text extract) │  │  (outlinks)     │  │ (SimHash)    │
         └──────────┬──────┘  └────────┬───────┘  └──────────────┘
                    │                  │
                    │          ┌───────▼──────────────────────┐
                    │          │  URL Deduplication           │
                    │          │  (Bloom filter: seen URLs)   │
                    │          │  New URLs → URL Frontier     │
                    │          └──────────────────────────────┘
                    │
         ┌──────────▼──────────────────────┐
         │  Document Store / Search Index  │
         └─────────────────────────────────┘
```

---

## How It Works

### 1. URL Frontier (Priority Queue)

The URL Frontier manages which URLs to crawl next:

```python
import heapq
import time
from collections import defaultdict

class URLFrontier:
    """
    Multi-queue frontier:
    - High-priority queue: fresh seed URLs, recently updated pages
    - Normal queue: scheduled re-crawls
    - Per-domain rate limiting: one request per second per domain
    """
    
    def __init__(self):
        self.pq = []  # (priority, url)
        self.domain_last_fetched = {}  # domain → last fetch timestamp
        self.domain_delay = 1.0  # seconds between requests to same domain
    
    def add(self, url: str, priority: float = 1.0):
        heapq.heappush(self.pq, (-priority, url))  # negate: higher score = higher priority
    
    def get_next(self) -> str | None:
        """
        Return next URL to crawl, respecting per-domain rate limits.
        Skip URLs whose domain was fetched too recently.
        """
        temp = []
        result = None
        now = time.time()
        
        while self.pq:
            priority, url = heapq.heappop(self.pq)
            domain = extract_domain(url)
            
            last = self.domain_last_fetched.get(domain, 0)
            if now - last >= self.domain_delay:
                # This domain is ready to crawl
                self.domain_last_fetched[domain] = now
                result = url
                break
            else:
                temp.append((priority, url))  # domain is cooling down, defer
        
        for item in temp:
            heapq.heappush(self.pq, item)
        
        return result

def prioritize_url(url: str, page_rank: float, last_modified: datetime | None,
                   change_frequency: float) -> float:
    """
    Priority = f(PageRank, freshness, how often the page changes)
    Higher priority → crawled sooner
    """
    freshness = 1.0
    if last_modified:
        age_days = (datetime.now() - last_modified).days
        freshness = max(0.1, 1.0 - age_days / 30)  # decay over 30 days
    
    return page_rank * freshness * change_frequency
```

### 2. Politeness — robots.txt

Respecting website crawl policies is legally and ethically required:

```python
from urllib.robotparser import RobotFileParser
import urllib.request

class RobotsCache:
    """Cache parsed robots.txt per domain (with 24-hour TTL)"""
    
    def __init__(self):
        self.cache = {}      # domain → (expires_at, RobotFileParser)
    
    def is_allowed(self, url: str, user_agent: str = "Googlebot") -> bool:
        domain = extract_domain(url)
        
        if domain not in self.cache or self.cache[domain][0] < time.time():
            # Fetch and parse robots.txt
            robots_url = f"https://{domain}/robots.txt"
            rp = RobotFileParser()
            rp.set_url(robots_url)
            try:
                rp.read()
                self.cache[domain] = (time.time() + 86400, rp)  # 24h TTL
            except Exception:
                # If robots.txt inaccessible, treat as "allow all"
                return True
        
        _, rp = self.cache[domain]
        
        # Check if URL is allowed
        allowed = rp.can_fetch(user_agent, url)
        
        # Also honor Crawl-delay directive
        self.crawl_delay[domain] = rp.crawl_delay(user_agent) or 1.0
        
        return allowed

# Sample robots.txt:
# User-agent: *
# Disallow: /admin/
# Disallow: /private/
# Crawl-delay: 2
#
# User-agent: Googlebot
# Allow: /
# Crawl-delay: 1
# Sitemap: https://example.com/sitemap.xml
```

### 3. Fetcher

```python
import aiohttp
import asyncio
from typing import Optional

class Fetcher:
    def __init__(self, robots_cache: RobotsCache, timeout: float = 10.0):
        self.robots = robots_cache
        self.timeout = timeout
        self.user_agent = "MyCrawler/1.0 (+https://example.com/bot)"
    
    async def fetch(self, url: str) -> Optional[dict]:
        # Check robots.txt
        if not self.robots.is_allowed(url, self.user_agent):
            return None  # Skip disallowed URLs
        
        # Check URL already seen (Bloom filter)
        if url_bloom_filter.contains(url):
            return None
        url_bloom_filter.add(url)
        
        headers = {
            "User-Agent": self.user_agent,
            "Accept": "text/html,application/xhtml+xml",
            "Accept-Encoding": "gzip, br"
        }
        
        async with aiohttp.ClientSession() as session:
            try:
                async with session.get(url, 
                                        headers=headers,
                                        timeout=aiohttp.ClientTimeout(total=self.timeout),
                                        allow_redirects=True,
                                        max_redirects=5) as resp:
                    
                    # Only process HTML (skip images, PDFs, etc.)
                    content_type = resp.headers.get("Content-Type", "")
                    if "text/html" not in content_type:
                        return None
                    
                    # Size limit: skip pages > 10 MB
                    content_length = int(resp.headers.get("Content-Length", 0))
                    if content_length > 10 * 1024 * 1024:
                        return None
                    
                    html = await resp.text(errors="replace")
                    
                    return {
                        "url": str(resp.url),     # final URL after redirects
                        "original_url": url,
                        "status_code": resp.status,
                        "html": html,
                        "headers": dict(resp.headers),
                        "fetched_at": time.time(),
                        "size_bytes": len(html.encode())
                    }
            
            except asyncio.TimeoutError:
                self.reschedule_later(url, delay=3600)  # retry in 1 hour
                return None
            except Exception as e:
                return None  # DNS failure, connection refused, etc.
```

### 4. URL Deduplication with Bloom Filter

With 5 billion URLs, you need space-efficient deduplication:

```python
from bitarray import bitarray
import hashlib
import math

class BloomFilter:
    """
    Probabilistic data structure: test if URL has been seen before.
    False positive rate: ~1% at optimal k for given n and m.
    No false negatives: if "not in filter", URL definitely hasn't been seen.
    """
    
    def __init__(self, n: int, fp_rate: float = 0.01):
        """
        n: expected number of elements (e.g., 5 billion)
        fp_rate: desired false positive rate
        """
        # Optimal bit array size: m = -n * ln(p) / (ln2)^2
        self.m = int(-n * math.log(fp_rate) / (math.log(2) ** 2))
        
        # Optimal number of hash functions: k = (m/n) * ln(2)
        self.k = int((self.m / n) * math.log(2))
        
        self.bits = bitarray(self.m)
        self.bits.setall(0)
        
        print(f"Bloom filter: {self.m / 8 / 1e9:.1f} GB, {self.k} hash functions")
        # For 5B URLs at 1% FP rate: ~7.2 GB, 7 hash functions
    
    def _hashes(self, url: str) -> list[int]:
        """Generate k independent hash positions"""
        results = []
        for i in range(self.k):
            h = hashlib.sha256(f"{i}:{url}".encode()).hexdigest()
            results.append(int(h, 16) % self.m)
        return results
    
    def add(self, url: str):
        for pos in self._hashes(url):
            self.bits[pos] = 1
    
    def contains(self, url: str) -> bool:
        return all(self.bits[pos] for pos in self._hashes(url))

# Memory requirements for 5 billion URLs:
# 3-replica dict: 5B * (100 bytes avg URL + overhead) = 500 GB RAM — infeasible
# Bloom filter: ~7.2 GB for 5B URLs at 1% FP rate — fits in RAM!
```

### 5. Near-Duplicate Detection (SimHash)

Two different URLs may serve almost identical content (e.g., `?page=1` vs `?page=2` with same content):

```python
import hashlib

def simhash(text: str, hash_bits: int = 64) -> int:
    """
    SimHash: generates a hash where similar documents have similar hashes.
    (Hamming distance between hashes ≈ difference between documents)
    """
    # Tokenize into n-grams
    tokens = text.lower().split()
    
    # Initialize vector of hash_bits dimensions
    v = [0] * hash_bits
    
    for token in tokens:
        # Hash each token
        token_hash = int(hashlib.md5(token.encode()).hexdigest(), 16)
        
        # For each bit position:
        # if bit is 1 in token_hash: increment v[i]
        # if bit is 0: decrement v[i]
        for i in range(hash_bits):
            if token_hash & (1 << i):
                v[i] += 1
            else:
                v[i] -= 1
    
    # Convert vector back to hash
    result = 0
    for i in range(hash_bits):
        if v[i] > 0:
            result |= (1 << i)
    
    return result

def hamming_distance(h1: int, h2: int) -> int:
    """Count differing bits between two SimHashes"""
    xor = h1 ^ h2
    return bin(xor).count('1')

def is_near_duplicate(hash1: int, hash2: int, threshold: int = 3) -> bool:
    """
    Documents are near-duplicates if Hamming distance ≤ 3
    (differ in ≤ 3 of 64 bit positions)
    """
    return hamming_distance(hash1, hash2) <= threshold

# Usage:
page1_hash = simhash("Apple sells iPhones and MacBooks worldwide")
page2_hash = simhash("Apple sells iPhones and MacBook worldwide")  # tiny difference
print(hamming_distance(page1_hash, page2_hash))  # → 1 or 2: near-duplicate!
```

### 6. Link Extraction and Normalization

```python
from bs4 import BeautifulSoup
from urllib.parse import urljoin, urlparse, urlunparse
import re

def extract_links(html: str, base_url: str) -> list[str]:
    """Parse HTML and extract all outbound links, normalized"""
    soup = BeautifulSoup(html, "html.parser")
    links = set()
    
    for tag in soup.find_all("a", href=True):
        href = tag["href"].strip()
        
        # Skip non-HTTP links
        if href.startswith(("mailto:", "javascript:", "tel:", "#")):
            continue
        
        # Resolve relative URLs: "/about" → "https://example.com/about"
        absolute_url = urljoin(base_url, href)
        
        # Normalize the URL
        normalized = normalize_url(absolute_url)
        if normalized:
            links.add(normalized)
    
    return list(links)

def normalize_url(url: str) -> str | None:
    """Canonicalize URL to avoid duplicates"""
    try:
        parsed = urlparse(url)
        
        # Only HTTP/HTTPS
        if parsed.scheme not in ("http", "https"):
            return None
        
        # Lowercase hostname
        hostname = parsed.netloc.lower()
        
        # Remove default port (http:443, https:80)
        if hostname.endswith(":80") and parsed.scheme == "http":
            hostname = hostname[:-3]
        if hostname.endswith(":443") and parsed.scheme == "https":
            hostname = hostname[:-4]
        
        # Normalize path (remove ./ and ../)
        path = parsed.path or "/"
        
        # Remove URL fragments (#section — server doesn't handle them)
        fragment = ""
        
        # Sort query parameters (a=1&b=2 == b=2&a=1)
        query = "&".join(sorted(parsed.query.split("&"))) if parsed.query else ""
        
        # Remove common tracking params
        tracking_params = {"utm_source", "utm_medium", "utm_campaign", 
                          "utm_content", "fbclid", "gclid"}
        params = [p for p in query.split("&") 
                  if p.split("=")[0] not in tracking_params and p]
        query = "&".join(params)
        
        return urlunparse((parsed.scheme, hostname, path, "", query, fragment))
    
    except Exception:
        return None
```

---

## Scale-Out Architecture

```
Distributed Crawler (100 machines):

1. URL Frontier sharded by domain hash:
   - Machine 1: domains a-m
   - Machine 2: domains n-z
   - Each machine owns URL queues for its domains

2. Fetcher workers per machine:
   - 1,000 async coroutines per machine × 100 machines = 100,000 concurrent fetches
   - Target: 100K pages/sec = 1 billion/day (with retries, some headroom needed)

3. Crawl state stored in:
   - Bloom filter: distributed (Redis Cluster or custom)
   - URL frontier: Redis sorted sets per domain (ZADD priority)
   - Raw HTML: streamed to S3 as fast as possible

4. Parser pool (separate from fetchers):
   - Fetchers: I/O bound (network) — lots of parallelism
   - Parsers: CPU bound (HTML parsing, SimHash) — 10 per machine, process queue
   
   Kafka topic "crawled.pages" → Parser workers → extract links + text → S3 + index
```

---

## Key Design Decisions

| Component | Choice | Reasoning |
|---|---|---|
| URL dedup | Bloom filter (7 GB for 5B URLs) | Hash set for 5B URLs = 500+ GB; Bloom filter: 7 GB with 1% FP |
| Near-dup detection | SimHash (64-bit) | Compare hashes, not full documents; O(1) per comparison |
| Frontier storage | Redis sorted set per domain | Priority queue with O(log N) push/pop; per-domain rate limiting |
| Crawl order | Priority by PageRank + freshness | Important pages crawled more often; stale pages re-crawled sooner |
| Max crawl depth | 5-10 hops from seed | Deeper links are lower quality; limit graph traversal depth |

---

## Hard-Learned Engineering Lessons

1. **Not addressing politeness**: sending 1,000 requests/second to a single domain will get your IPs banned and is unethical. Per-domain rate limiting (1 req/sec default) is required.

2. **Forgetting robots.txt**: not mentioning `robots.txt` is a significant gap. Crawlers must check it before fetching any URL from a domain.

3. **Using a hash set for URL deduplication**: a Python `set` of 5 billion URLs requires hundreds of GB of RAM. Bloom filter is the correct answer.

4. **Not handling redirects/cycles**: the same content can live at multiple URLs (redirects, query param variants). URL normalization and redirect tracking prevent re-crawling duplicates.

5. **Ignoring content type filtering**: not all URLs contain HTML. Without checking Content-Type, you'd store gigabytes of PDFs, images, and binary files.

6. **Missing the spider trap**: websites can programmatically generate infinite URLs (`/page/1`, `/page/2`, ..., `/page/∞`). Limit crawl depth and count of URLs per domain.

7. **Not discussing re-crawl scheduling**: a crawler isn't a one-time job — pages change. Freshness-aware scheduling (re-crawl frequently updated news sites hourly; rarely updated pages monthly) is a key design dimension.

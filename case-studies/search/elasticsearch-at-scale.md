# Design a Distributed Search System (Elasticsearch at Scale)

## Problem Statement

Design a distributed full-text search system capable of indexing and querying billions of documents with sub-second latency. This models how Elasticsearch works internally and how systems like GitHub's code search, e-commerce product search, and log analytics are built.

**Scale requirements**:
- 10 billion documents indexed
- 50 TB of index data
- 50,000 search queries per second
- Index a new document within 1 second of ingestion
- p99 search latency < 100ms

---

## Core Concepts

### Inverted Index (Foundation of Text Search)

```
Documents:
  Doc 1: "The quick brown fox jumps over the lazy dog"
  Doc 2: "A quick brown dog outpaced a lazy fox"
  Doc 3: "The dog is not lazy, it is slow"

Inverted Index (term → posting list):
  "quick" → [Doc1: positions=[2], Doc2: positions=[3]]
  "brown" → [Doc1: positions=[3], Doc2: positions=[4]]
  "dog"   → [Doc1: positions=[8], Doc2: positions=[5], Doc3: positions=[2]]
  "lazy"  → [Doc1: positions=[9], Doc2: positions=[7], Doc3: positions=[4]]
  "fox"   → [Doc1: positions=[4], Doc2: positions=[8]]

Query: "quick brown fox"
  → AND: docs containing all three terms
  → Doc1 and Doc2 match → ranked by TF-IDF
  
Phrase query: "lazy dog" (must be adjacent)
  → Check Doc1: "lazy"@9, "dog"@8 — positions differ by 1? No (9-8=1 ✓)
  → Doc1 matches "lazy dog"
  → Doc2: "lazy"@7, "dog"@5 — 7-5=2 ✗ no match
```

### Lucene (Elasticsearch's underlying engine)

Elasticsearch is built on **Apache Lucene**, which provides:
- Inverted index (per shard)
- BM25 scoring (improved TF-IDF)
- Compressed posting lists (for-each, skip lists)
- Segment-based storage (immutable segments + merge)

---

## Architecture

### Cluster Architecture

```
┌────────────────────────────────────────────────────────────────┐
│  Elasticsearch Cluster                                         │
│                                                                │
│  Master Node: cluster state, shard allocation, index mapping   │
│  (dedicated, not for data)                                      │
│                                                                │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │ Data Node 1  │  │ Data Node 2  │  │ Data Node 3  │         │
│  │              │  │              │  │              │         │
│  │  Shard P0    │  │  Shard P1    │  │  Shard P2    │         │
│  │  Shard R1    │  │  Shard R2    │  │  Shard R0    │         │
│  └──────────────┘  └──────────────┘  └──────────────┘         │
│                                                                │
│  P = Primary shard, R = Replica shard                         │
│  Each shard is a complete Lucene index                         │
└────────────────────────────────────────────────────────────────┘
```

### Sharding Strategy

```python
# Documents distributed across shards by routing key
def routing_shard(doc_id: str, num_primary_shards: int) -> int:
    # Default: hash(doc_id or routing key) % num_shards
    return hash(doc_id) % num_primary_shards

# Example: 3 primary shards
# doc-001 → shard 0
# doc-002 → shard 1  
# doc-999 → shard 2

# WARNING: num_primary_shards is FIXED at index creation time.
# Changing it requires full reindex. Choose carefully.
# Rule of thumb: aim for 20-50 GB per shard.
# 50 TB / 20 GB = 2,500 primary shards if evenly distributed.
```

---

## How it works

### 1. Document Indexing

```
POST /products/_doc/12345
{
  "name": "Apple iPhone 15 Pro",
  "description": "Titanium design, A17 Pro chip, 48MP camera",
  "price": 999.00,
  "category": "electronics",
  "brand": "Apple",
  "in_stock": true
}
```

**Indexing pipeline (per primary shard)**:
```
1. Route to primary shard: shard = hash("12345") % num_shards
2. Primary shard:
   a. Parse document, apply mapping (field types)
   b. Analyze text fields:
      "Apple iPhone 15 Pro"
      → Tokenize: ["apple", "iphone", "15", "pro"]
      → Lowercase filter: ["apple", "iphone", "15", "pro"]
      → (optional) Stop word removal, stemming
   c. Write to in-memory segment (Lucene "translog")
   d. Return ACK to client (durability via translog before fsync)
3. Replicate to replica shards (async, default 1 replica)
4. Background: refresh (every 1 second): make docs searchable
   → Lucene flush: in-memory buffer → new on-disk segment
```

### 2. Lucene Segments (Immutable Write Model)

```
Each shard = multiple Lucene segments (immutable files)

Time 0:  [segment-1: docs 1-100]
Time 1s: [segment-1: docs 1-100] [segment-2: docs 101-200]
Time 2s: [segment-1] [segment-2] [segment-3: docs 201-300]

Search: query ALL segments, merge results

Merge policy:
  When many small segments accumulate → merge into larger segment
  (reduces per-search overhead of searching many small segments)
  Merges happen in background; old segments deleted after merge

Deletes:
  Lucene segments are immutable.
  Delete = set tombstone bit in .del file
  Actual deletion: happens at merge time (deleted docs excluded from merged segment)

Updates = Delete + Insert
```

### 3. Search Execution (Scatter-Gather)

```
User searches: "iPhone camera review"

1. Coordinating node receives request
   
2. Scatter: send query to all relevant shards (all 3 here, for non-routed query)
   Each shard independently:
     a. Analyze query: ["iphone", "camera", "review"]
     b. Lookup posting lists for each term
     c. Compute BM25 score per document
     d. Return top-K (e.g., 10) doc IDs + scores to coordinator
   
3. Gather: coordinator receives top-10 from each of 3 shards
   → Merge-sort → global top-10 by score
   
4. Fetch: coordinator requests full documents from shards (just IDs in step 2)
   (fetch phase: retrieve _source fields for final results)
   
5. Return final results with source documents to client
```

**BM25 Scoring** (Elasticsearch's default since 5.0):
$$\text{BM25}(t, d) = \text{IDF}(t) \times \frac{\text{TF}(t,d) \times (k_1 + 1)}{\text{TF}(t,d) + k_1 \times (1 - b + b \times \frac{|d|}{avgdl})}$$

Where $k_1=1.2$, $b=0.75$ — length normalization prevents long documents from dominating.

### 4. Index Mapping

Define field types explicitly to control analysis and storage:

```json
PUT /products
{
  "settings": {
    "number_of_shards": 5,
    "number_of_replicas": 1,
    "index.refresh_interval": "1s"
  },
  "mappings": {
    "properties": {
      "name": {
        "type": "text",
        "analyzer": "english",
        "fields": {
          "keyword": {"type": "keyword"}  // for exact match and aggregations
        }
      },
      "description": {"type": "text", "analyzer": "english"},
      "price": {"type": "float"},
      "category": {"type": "keyword"},   // no text analysis: exact match + aggregations
      "brand": {"type": "keyword"},
      "in_stock": {"type": "boolean"},
      "embedding": {                     // for vector/semantic search
        "type": "dense_vector",
        "dims": 768,
        "index": true,
        "similarity": "cosine"
      }
    }
  }
}
```

### 5. Aggregations (Analytics Use Case)

Elasticsearch is also used for log analytics (ELK stack). Aggregations run on non-text fields:

```json
// "How many products per category, and what's the average price?"
GET /products/_search
{
  "query": {"match": {"brand": "Apple"}},
  "aggs": {
    "by_category": {
      "terms": {"field": "category", "size": 10},
      "aggs": {
        "avg_price": {"avg": {"field": "price"}}
      }
    }
  }
}

// Response:
// "electronics": count=150, avg_price=$899
// "accessories": count=45, avg_price=$49
```

Aggregations work on `keyword` and numeric fields (not analyzed text) using columnar data structures (doc values) stored to disk per segment.

### 6. Vector Search (Semantic Search)

Modern Elasticsearch (7.3+) supports dense vector search for semantic similarity:

```python
from sentence_transformers import SentenceTransformer

model = SentenceTransformer('all-MiniLM-L6-v2')

# At index time: compute embedding
doc = {"name": "Apple iPhone 15 Pro", "description": "..."}
doc["embedding"] = model.encode(doc["name"]).tolist()  # 384-dim vector

# Index document
es.index(index="products", id="12345", document=doc)

# At query time: semantic search
query_vector = model.encode("best smartphone camera").tolist()

results = es.search(
    index="products",
    knn={
        "field": "embedding",
        "query_vector": query_vector,
        "k": 10,          # return top 10
        "num_candidates": 100  # ANN search over 100 candidates per shard
    }
)
```

Elasticsearch uses **HNSW** (Hierarchical Navigable Small World) for approximate nearest neighbor vector search.

**Hybrid search** (combine text + vector):

```json
{
  "retriever": {
    "rrf": {  // Reciprocal Rank Fusion
      "retrievers": [
        {"standard": {"query": {"match": {"name": "iPhone camera"}}}},
        {"knn": {"field": "embedding", "query_vector": [...], "k": 10}}
      ]
    }
  }
}
```

---

## Key Design Decisions

| Decision | Choice | Reasoning |
|---|---|---|
| Shard count | ~20-50 GB per shard target | Balance: too few = hot shards; too many = overhead |
| Refresh interval | 1 second (default) | Near-real-time search; increase to 30s for write-heavy batch ingestion |
| Bulk indexing | Bulk API in batches of 5-10 MB | Single-doc indexing ~10x slower than bulk |
| Scoring | BM25 (default) | Better than TF-IDF for varying document lengths |
| Field types | keyword vs text | keyword for filtering/aggregations; text for full-text search |

---

## Common Pitfalls

- **Too many shards**: 1,000 shards for 100 GB of data is over-sharding. Each shard has overhead (CPU, memory, file handles). Target 20–50 GB/shard.
- **Dynamic mapping explosions**: ES auto-creates field mappings. A log index with thousands of unique JSON keys creates thousands of field mappings — crashes the master node. Use strict mapping or flatten logs.
- **Ignoring the refresh interval**: the default 1-second refresh means `index → searchable` has 1-second latency. If you need immediate searchability, call `_refresh` — but this creates many small segments (hurts search performance).
- **Using _source for aggregations**: aggregations use `doc_values` (columnar on-disk storage), not `_source`. Disabling `doc_values` on numeric/keyword fields breaks aggregations.

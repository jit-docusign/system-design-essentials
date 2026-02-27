# Search Engines (Elasticsearch / OpenSearch)

## What it is

A **search engine database** is a system built around an **inverted index** — a data structure that maps terms to the documents containing them — enabling fast full-text search, relevance ranking, faceted filtering, and aggregations over large collections of documents.

Search engines are not general-purpose databases. They are purpose-built for **text-oriented read queries** with ranking, and they sacrifice strict consistency and transactional writes for query power and horizontal read scalability.

**Common systems**: Elasticsearch, OpenSearch (AWS fork of Elasticsearch), Apache Solr, Typesense, Meilisearch, Algolia (hosted).

---

## How it works

### Inverted Index

The core data structure of a search engine:

```
Forward index (document → words):
  Doc 1: "the quick brown fox"
  Doc 2: "jump over the lazy dog"
  Doc 3: "the brown dog sleeps"

Inverted index (word → documents):
  "the"   → [Doc1, Doc2, Doc3]
  "brown" → [Doc1, Doc3]
  "dog"   → [Doc2, Doc3]
  "fox"   → [Doc1]
  "quick" → [Doc1]
  "lazy"  → [Doc2]
  "jump"  → [Doc2]
  "over"  → [Doc2]
  "sleeps"→ [Doc3]
```

A query for "brown dog" looks up "brown" → {Doc1, Doc3} and "dog" → {Doc2, Doc3}, then intersects to find {Doc3} as the top result.

At production scale, each entry stores not just the document ID but also:
- **Term frequency (TF)**: how many times the term appears in the document.
- **Positions**: character offsets and word positions for phrase proximity scoring.
- **Stored fields**: original document content for retrieval.

### Text Analysis Pipeline

Before indexing, text goes through an **analysis chain**:
```
Input: "The Quick Brown Fox Jumped!"

1. Tokenizer:     ["The", "Quick", "Brown", "Fox", "Jumped"]
2. Lowercase:     ["the", "quick", "brown", "fox", "jumped"]
3. Stop filter:   ["quick", "brown", "fox", "jumped"]  (removed "the")
4. Stemmer:       ["quick", "brown", "fox", "jump"]    (jumped → jump)
```

The same analysis runs on the query, so "Jumping over Foxes" matches documents with "jumped" and "fox". The analyzer is configurable per field.

### Indexing Documents

```json
// PUT /products/_doc/1
{
  "id": 1,
  "title": "Mechanical Keyboard with RGB Lighting",
  "description": "Cherry MX Brown switches, TKL layout, USB-C",
  "category": "Peripherals",
  "brand": "Keychron",
  "price": 149.99,
  "rating": 4.7,
  "in_stock": true,
  "tags": ["keyboard", "mechanical", "rgb", "gaming"]
}
```

### Searching

```json
// GET /products/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "title": "mechanical keyboard" }}   // full-text on title
      ],
      "filter": [
        { "term":  { "in_stock": true }},               // exact match (no scoring)
        { "range": { "price": { "gte": 50, "lte": 300 }}}
      ]
    }
  },
  "sort": [
    { "_score": "desc" },   // relevance first
    { "rating": "desc" }    // then by rating
  ],
  "aggregations": {
    "brands": {
      "terms": { "field": "brand" }       // facet: count by brand
    },
    "avg_price": {
      "avg": { "field": "price" }         // metric aggregation
    }
  }
}
```

### Relevance Scoring (BM25)

Elasticsearch uses **BM25** (Best Match 25) — an improved TF-IDF formula — to score how well a document matches a query:

$$\text{score}(d, q) = \sum_{t \in q} \text{IDF}(t) \cdot \frac{f(t, d) \cdot (k_1 + 1)}{f(t, d) + k_1 \cdot (1 - b + b \cdot |d| / \text{avgdl})}$$

Where:
- $f(t, d)$ = term frequency in document
- $|d|$ = document length
- $\text{avgdl}$ = average document length
- $k_1$ = term frequency saturation (default 1.2)
- $b$ = length normalization (default 0.75)

Longer documents are normalized to prevent artificially high scores due to term repetition.

Score can be boosted by field: `title` matches boosted 3× over `description` matches.

### Sharding and Distribution

Elasticsearch distributes data across **shards** (Lucene indexes):
- Each index is split into **primary shards** (default 1–5).
- Each primary shard has **replica shards** for availability and read scaling.
- Writes go to the primary shard; reads can be served by any primary or replica.

```
Index: products (3 primary shards, 1 replica each)
Node 1: Shard 0 [primary], Shard 1 [replica]
Node 2: Shard 1 [primary], Shard 2 [replica]
Node 3: Shard 2 [primary], Shard 0 [replica]
```

**Near-real-time (NRT) search**: documents are indexed into an in-memory buffer and flushed to a searchable Lucene segment every 1 second (the **refresh interval**). Documents are not immediately searchable — there's a ~1 second visibility delay by default.

### Common Use Cases

- **Product search with facets**: filtering by category, price range, brand with result counts per facet.
- **Full-text document search**: searching support articles, legal documents, codebases.
- **Log analytics** (Elastic Stack / ELK): Elasticsearch as the log store, Kibana for dashboards.
- **Autocomplete / typeahead**: edge ngram analyzer on a search field indexes partial words ("mecha" → matches "mechanical").
- **Geospatial search**: geo_point fields enable "find nearest X within Y km" queries.

---

## Key Trade-offs

| Advantage | Disadvantage |
|---|---|
| Fast full-text search with relevance ranking | Not a source of truth — sync from primary DB required |
| Rich faceting and aggregation | Near-real-time, not consistent (1s delay) |
| Horizontal read scaling | High resource usage (memory-intensive) |
| Schema-flexible JSON documents | Complex to operate and tune at scale |
| Built-in distributed architecture | Data can drift from primary source |

---

## When to use

Use a search engine when:
- Users type free-form text and expect **relevant results** with ranking (not just exact matches).
- You need **faceted filtering** with item counts per filter value.
- You need **full-text search** across large bodies of text.
- You need **log analytics** at scale (ELK stack).
- You need **autocomplete** and fuzzy matching.

Do not use a search engine as your primary database. Always maintain a primary source of truth (relational or document DB) and sync to Elasticsearch for search. Elasticsearch is optimized for reads, not transactional writes.

---

## Common Pitfalls

- **Using Elasticsearch as the primary database**: Elasticsearch prioritizes availability and search performance over consistency. Data loss on unclean shutdown is possible. Always have a primary source of truth and treat Elasticsearch as a derived read model.
- **Too many shards**: shards have overhead (JVM heap per shard, per-shard heartbeats). Avoid over-sharding; start with fewer larger shards and add only when needed.
- **Mapping explosion**: if you store dynamic or user-defined JSON fields, Elasticsearch may create thousands of mappings → heap pressure. Enable `dynamic: strict` and define mappings explicitly.
- **Not handling index refresh for real-time needs**: if a search page must reflect a write immediately, explicitly `POST /index/_refresh` after the write (but this is expensive — avoid in hot write paths).
- **Ignoring replica overhead**: replicas improve read throughput and availability, but every write must also be replicated. Heavy write loads with many replicas increase write latency.

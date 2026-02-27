# System Design Essentials

A structured, cloud-agnostic reference for understanding the core building blocks and design patterns that underpin large-scale distributed systems. The goal is deep conceptual understanding — not tool-specific tutorials.

---

## Table of Contents

- [Section 1 — Foundations of System Design](#section-1--foundations-of-system-design)
  - [1. Core Concepts & Trade-offs](#1-core-concepts--trade-offs)
  - [2. Reliability & Availability](#2-reliability--availability)
  - [3. Networking & Protocols](#3-networking--protocols)
  - [4. Traffic Management](#4-traffic-management)
  - [5. Relational Databases](#5-relational-databases)
  - [6. Non-Relational Databases](#6-non-relational-databases)
  - [7. Caching](#7-caching)
  - [8. Scalability Patterns](#8-scalability-patterns)
  - [9. Asynchronous Processing & Messaging](#9-asynchronous-processing--messaging)
  - [10. Microservices & Service Architecture](#10-microservices--service-architecture)
  - [11. Storage Infrastructure](#11-storage-infrastructure)
  - [12. Distributed Systems Internals](#12-distributed-systems-internals)
  - [13. Security](#13-security)
  - [14. Observability](#14-observability)
- [Section 2 — Case Studies](#section-2--case-studies)
  - [A. Social Media & Feeds](#a-social-media--feeds)
  - [B. Messaging & Real-time Communication](#b-messaging--real-time-communication)
  - [C. Video & Media Streaming](#c-video--media-streaming)
  - [D. Location & Transportation](#d-location--transportation)
  - [E. Search & Discovery](#e-search--discovery)
  - [F. Storage & File Sync](#f-storage--file-sync)
  - [G. E-commerce & Payments](#g-e-commerce--payments)
  - [H. Infrastructure & Utilities](#h-infrastructure--utilities)
  - [I. Analytics & Data Systems](#i-analytics--data-systems)
  - [J. Gaming & Leaderboards](#j-gaming--leaderboards)
- [Contributing](#contributing)

---

## Section 1 — Foundations of System Design

---

### 1. Core Concepts & Trade-offs

> The mental models and trade-off frameworks that underpin every design decision.

| Topic | Description | Key Sub-concepts |
|---|---|---|
| [Performance vs Scalability](foundations/core-concepts/performance-vs-scalability.md) | Performance = speed for one user; scalability = maintaining speed under increasing load | Response time degradation, load testing, vertical vs horizontal bottlenecks |
| [Latency vs Throughput](foundations/core-concepts/latency-vs-throughput.md) | Latency is the time per operation; throughput is operations per unit time | P50/P99 percentiles, Little's Law, queue theory basics |
| [Availability vs Consistency](foundations/core-concepts/availability-vs-consistency.md) | The central tension in any distributed system — you often can't have both at peak | Eventual consistency, strong consistency, read-your-writes |
| [CAP Theorem](foundations/core-concepts/cap-theorem.md) | A distributed system can guarantee at most 2 of: Consistency, Availability, Partition Tolerance | CP vs AP systems, network partitions in practice |
| [PACELC Theorem](foundations/core-concepts/pacelc-theorem.md) | Extends CAP: even without partitions, systems trade off latency vs consistency | Why CAP alone is insufficient for real-world decisions |
| [ACID Properties](foundations/core-concepts/acid.md) | Atomicity, Consistency, Isolation, Durability — the guarantees relational databases provide | Transaction isolation levels, durability trade-offs |
| [BASE Properties](foundations/core-concepts/base.md) | Basically Available, Soft state, Eventually consistent — the NoSQL philosophy | When to accept eventual consistency, convergence |
| [Back-of-the-Envelope Estimation](foundations/core-concepts/estimation.md) | Rapid capacity planning using rough math to validate system feasibility | QPS, storage growth, bandwidth, read/write ratio, memory sizing |

---

### 2. Reliability & Availability

> Patterns for building systems that continue to operate correctly in the presence of failures.

| Topic | Description | Key Sub-concepts |
|---|---|---|
| [Consistency Patterns](foundations/reliability/consistency-patterns.md) | Strong, weak, and eventual consistency — when each is the right choice | Read-after-write, monotonic reads, causal consistency |
| [Failover Strategies](foundations/reliability/failover.md) | How systems switch to a standby when the primary fails | Active-passive vs active-active, failover detection lag, split-brain |
| [Replication](foundations/reliability/replication.md) | Copying data across nodes to enable fault tolerance and read scaling | Synchronous vs asynchronous replication, replication lag, multi-master conflicts |
| [Fault Tolerance](foundations/reliability/fault-tolerance.md) | Designing systems to continue operating despite component failures | Bulkheads, timeouts, retries with backoff, graceful degradation |
| [SLA / SLO / SLI](foundations/reliability/sla-slo-sli.md) | Defining and measuring reliability contracts in production | The "nines" (99.9%, 99.99%), error budgets, alerting thresholds |
| [Redundancy](foundations/reliability/redundancy.md) | Eliminating single points of failure across all layers of a system | N+1 redundancy, geographic redundancy, cold/warm/hot standby |

---

### 3. Networking & Protocols

> How data moves across a network and the protocols that govern communication.

| Topic | Description | Key Sub-concepts |
|---|---|---|
| [DNS (Domain Name System)](foundations/networking/dns.md) | Translates human-readable domain names into IP addresses | Hierarchical resolution, TTL and caching, authoritative vs recursive resolvers, DNS propagation |
| [HTTP / HTTPS](foundations/networking/http-https.md) | The foundational request-response protocol of the web | HTTP verbs, status codes, headers, keep-alive, HTTP/2, TLS handshake |
| [TCP vs UDP](foundations/networking/tcp-vs-udp.md) | TCP provides reliable ordered delivery; UDP sacrifices reliability for speed | 3-way handshake, flow control, congestion control, use cases for each |
| [OSI Model](foundations/networking/osi-model.md) | 7-layer conceptual framework for understanding network protocols | L4 (Transport) vs L7 (Application) — the most system-design-relevant boundary |
| [WebSockets](foundations/networking/websockets.md) | Full-duplex persistent connection enabling real-time bidirectional communication | Upgrade handshake, connection lifecycle, scaling WebSocket servers |
| [Long Polling & Server-Sent Events (SSE)](foundations/networking/long-polling-sse.md) | Server push approximations before or instead of WebSockets | Polling overhead, SSE semantics, tradeoffs vs WebSockets |
| [REST](foundations/networking/rest.md) | Stateless, resource-oriented API architecture style over HTTP | Uniform interface, statelessness, HATEOAS, REST vs RPC distinctions |
| [RPC / gRPC](foundations/networking/rpc-grpc.md) | Remote procedure calls for efficient communication between internal services | Protobuf serialization, streaming, service definitions, code generation |
| [GraphQL](foundations/networking/graphql.md) | Query language for APIs enabling flexible client-driven data fetching | Schema, resolvers, N+1 problem, batching, subscriptions |

---

### 4. Traffic Management

> How to distribute, control, and protect traffic flowing into and between services.

| Topic | Description | Key Sub-concepts |
|---|---|---|
| [Load Balancers (L4 vs L7)](foundations/traffic/load-balancers.md) | L4 routes by IP/port; L7 routes by content, headers, or URL path | Health checks, session persistence (sticky sessions), SSL termination |
| [Load Balancing Algorithms](foundations/traffic/load-balancing-algorithms.md) | Strategies for distributing requests across a pool of servers | Round robin, least connections, IP hash, weighted, consistent hashing |
| [Reverse Proxy](foundations/traffic/reverse-proxy.md) | A server that sits in front of backends to centralize concerns | TLS offloading, compression, caching, request buffering |
| [API Gateway](foundations/traffic/api-gateway.md) | Single entry point for clients to reach microservices | Authentication, rate limiting, routing, protocol translation, SSL termination |
| [Rate Limiting](foundations/traffic/rate-limiting.md) | Controlling request frequency to prevent abuse, overload, and DoS attacks | Token bucket, leaky bucket, fixed window, sliding window log/counter |
| [Circuit Breaker](foundations/traffic/circuit-breaker.md) | Detects and isolates failing downstream services to prevent cascading failures | Closed/Open/Half-open states, failure thresholds, fallback strategies |
| [Content Delivery Network (CDN)](foundations/traffic/cdn.md) | Geographically distributed edge servers that cache and serve content closer to users | Edge PoPs, origin shielding, Push vs Pull CDN, cache invalidation, TTL |

---

### 5. Relational Databases

> Structured, schema-based storage with strong consistency and rich query capabilities.

| Topic | Description | Key Sub-concepts |
|---|---|---|
| [Relational Databases (RDBMS)](foundations/databases/rdbms.md) | Schema-based tabular storage with SQL and full ACID guarantees | Normalization, joins, foreign keys, constraints |
| [Indexing](foundations/databases/indexing.md) | Data structures that accelerate read queries at the cost of write overhead | B-tree vs hash indexes, composite indexes, covering indexes, index cardinality |
| [Database Replication](foundations/databases/db-replication.md) | Maintaining copies of data across multiple nodes for availability and read scaling | Primary-replica lag, replication factor, read replicas, failover promotion |
| [Database Sharding](foundations/databases/sharding.md) | Horizontal partitioning — splitting a dataset across multiple database nodes | Shard keys, range vs hash sharding, resharding, cross-shard queries |
| [Database Federation](foundations/databases/federation.md) | Splitting databases by functional domain to reduce load per database | Per-service databases, cross-service joins, eventual consistency |
| [Denormalization](foundations/databases/denormalization.md) | Intentionally adding redundancy to avoid expensive joins in read-heavy systems | Trade-off between write complexity and read performance |
| [Transactions & Locking](foundations/databases/transactions-locking.md) | ACID semantics, concurrency control, and deadlock prevention | Isolation levels (Read Committed, Serializable), optimistic vs pessimistic locking, MVCC |
| [SQL Query Optimization](foundations/databases/sql-optimization.md) | Techniques to improve query performance at scale | EXPLAIN plans, query rewrites, partition pruning, avoiding N+1 queries |

---

### 6. Non-Relational Databases

> Storage systems optimized for specific data models, access patterns, or scale characteristics.

| Topic | Description | Key Sub-concepts |
|---|---|---|
| [NoSQL Overview & BASE](foundations/nosql/nosql-overview.md) | Non-relational storage models built for scale, flexibility, and high availability | Schema-on-read, BASE vs ACID, when to choose NoSQL |
| [Key-Value Stores](foundations/nosql/key-value-stores.md) | Hash-table abstraction with O(1) reads and writes | In-memory vs persistent, TTL, use cases (sessions, caching, leaderboards) |
| [Document Stores](foundations/nosql/document-stores.md) | JSON/BSON document storage with flexible, nested schema | Embedded vs referenced documents, secondary indexes, aggregation pipelines |
| [Wide-Column Stores](foundations/nosql/wide-column-stores.md) | Column-family model designed for massive scale and high write throughput | Partition key + clustering key, compaction, tombstones, tunable consistency |
| [Graph Databases](foundations/nosql/graph-databases.md) | Node and edge model for relationship-intensive data | Traversal queries, social graphs, fraud detection, OLTP vs OLAP graphs |
| [Time-Series Databases](foundations/nosql/time-series-databases.md) | Optimized for timestamped event data with high write throughput and range queries | Downsampling, retention policies, compression, metrics and IoT use cases |
| [Search Engines & Inverted Index](foundations/nosql/search-inverted-index.md) | Full-text search using inverted index structures | Tokenization, relevance scoring (TF-IDF / BM25), shards and replicas |
| [SQL vs NoSQL Decision](foundations/nosql/sql-vs-nosql.md) | Framework for choosing the right storage model for a given problem | Query patterns, consistency needs, schema flexibility, operational complexity |

---

### 7. Caching

> Storing computed or fetched results closer to where they are needed to reduce latency and backend load.

| Topic | Description | Key Sub-concepts |
|---|---|---|
| [Cache Layers](foundations/caching/cache-layers.md) | Where caches live in a system architecture | Client-side, CDN, web server, application-level, database query cache |
| [Cache-Aside (Lazy Loading)](foundations/caching/cache-aside.md) | Application reads from cache; on miss, fetches from DB and populates cache | Cold start, cache miss penalty, stale reads |
| [Write-Through](foundations/caching/write-through.md) | Writes go to cache and DB synchronously; always consistent | Higher write latency, no data loss risk |
| [Write-Behind (Write-Back)](foundations/caching/write-behind.md) | Writes go to cache first; DB is updated asynchronously | Lower write latency, risk of data loss on cache failure |
| [Refresh-Ahead](foundations/caching/refresh-ahead.md) | Proactively refreshes cache entries before they expire | Reduces miss latency for predictable access patterns |
| [Cache Eviction Policies](foundations/caching/eviction-policies.md) | Strategies for deciding which entries to remove when the cache is full | LRU, LFU, FIFO, sliding window, ARC |
| [Cache Invalidation](foundations/caching/cache-invalidation.md) | Ensuring stale data is removed or updated — widely considered the hardest caching problem | TTL-based expiry, event-driven invalidation, versioning / cache-busting |
| [Thundering Herd / Cache Stampede](foundations/caching/thundering-herd.md) | A burst of simultaneous requests hitting the origin when a popular cache entry expires | Probabilistic early expiration, request coalescing, mutex locks |
| [Distributed Caching](foundations/caching/distributed-caching.md) | Spreading a cache across multiple nodes to exceed single-machine limits | Consistent hashing, hot key problem, replication, coherence |

---

### 8. Scalability Patterns

> Architectural strategies for handling growth in users, data, and request volume.

| Topic | Description | Key Sub-concepts |
|---|---|---|
| [Vertical vs Horizontal Scaling](foundations/scalability/vertical-vs-horizontal.md) | Scale up (bigger machine) vs scale out (more machines) | Cost curves, ceiling of vertical scaling, stateless requirements for horizontal |
| [Stateless Architecture](foundations/scalability/stateless-architecture.md) | Externalizing all session/state to allow any server to handle any request | Session stores, JWTs, sticky sessions as an anti-pattern |
| [Consistent Hashing](foundations/scalability/consistent-hashing.md) | A hashing scheme that minimizes key redistribution when nodes are added or removed | Hash ring, virtual nodes, replication factor, hotspot mitigation |
| [Data Partitioning Strategies](foundations/scalability/data-partitioning.md) | Schemes for distributing data across nodes | Range, hash, directory-based, geographic partitioning; partition key selection |
| [CQRS (Command Query Responsibility Segregation)](foundations/scalability/cqrs.md) | Separating the read model from the write model to allow independent scaling | Event-driven sync between models, eventual consistency, read projections |
| [Event Sourcing](foundations/scalability/event-sourcing.md) | Storing system state as an immutable sequence of events rather than current state | Event log, replay, snapshots, audit trail, CQRS pairing |
| [Hotspot & Skew Avoidance](foundations/scalability/hotspot-avoidance.md) | Preventing uneven load across partitions due to high-traffic keys | Salting, compound keys, write spreading, celebrity problem |

---

### 9. Asynchronous Processing & Messaging

> Decoupling producers from consumers to improve resilience, scalability, and throughput.

| Topic | Description | Key Sub-concepts |
|---|---|---|
| [Message Queues](foundations/messaging/message-queues.md) | Buffer between producers and consumers for async, guaranteed delivery | At-least-once vs exactly-once, ordering, visibility timeouts, dead letter queues |
| [Pub/Sub Pattern](foundations/messaging/pub-sub.md) | Broadcast messages to multiple independent subscribers | Topics, fanout, decoupling, push vs pull delivery |
| [Event-Driven Architecture](foundations/messaging/event-driven.md) | Systems react to events rather than making synchronous calls | Event producers/consumers, event schema, idempotency, choreography vs orchestration |
| [Log-Based Messaging (Kafka)](foundations/messaging/log-based-messaging.md) | A durable, ordered, replayable commit log as the messaging backbone | Partitions, consumer groups, offsets, retention, compaction |
| [Dead Letter Queues](foundations/messaging/dead-letter-queues.md) | Handling failed messages without blocking the main processing pipeline | Retry limits, poison messages, alerting, reprocessing strategies |
| [Back Pressure](foundations/messaging/back-pressure.md) | Preventing fast producers from overwhelming slow consumers | Reactive streams, bounded buffers, shedding load gracefully |
| [Stream vs Batch Processing](foundations/messaging/stream-vs-batch.md) | Real-time per-event processing vs periodic bulk processing of accumulated data | Micro-batching, windowing, Lambda architecture, Kappa architecture |

---

### 10. Microservices & Service Architecture

> Decomposing systems into independently deployable, loosely coupled services.

| Topic | Description | Key Sub-concepts |
|---|---|---|
| [Monolith vs Microservices](foundations/microservices/monolith-vs-microservices.md) | Trade-offs between deployment simplicity and independent scalability | Modular monolith, service boundaries, distributed system complexity |
| [Service Discovery](foundations/microservices/service-discovery.md) | How services locate each other dynamically at runtime | Client-side vs server-side discovery, service registry (Consul, etcd, Zookeeper) |
| [Service Mesh](foundations/microservices/service-mesh.md) | Infrastructure layer for service-to-service communication concerns | Sidecar proxy, mTLS, observability, traffic policies |
| [API Gateway Patterns](foundations/microservices/api-gateway-patterns.md) | Advanced gateway usage beyond simple routing | BFF (Backend For Frontend), aggregation, protocol translation |
| [Sidecar Pattern](foundations/microservices/sidecar-pattern.md) | Deploying cross-cutting concerns alongside services in a co-located process | Logging, auth, service mesh proxy, co-location without coupling |
| [Saga Pattern](foundations/microservices/saga-pattern.md) | Managing distributed transactions through a sequence of compensating actions | Choreography vs orchestration, rollback semantics, eventual consistency |
| [Strangler Fig Pattern](foundations/microservices/strangler-fig.md) | Incrementally migrating a monolith to microservices without a big-bang rewrite | Feature flags, parallel running, traffic migration |

---

### 11. Storage Infrastructure

> Low-level storage systems and how large-scale data is physically organized and replicated.

| Topic | Description | Key Sub-concepts |
|---|---|---|
| [Object Storage](foundations/storage/object-storage.md) | Flat key-value store for unstructured binary data (images, videos, backups) | Immutability, eventual consistency, presigned URLs, multipart upload |
| [Block Storage](foundations/storage/block-storage.md) | Low-level addressable storage exposed as volumes to operating systems | IOPS, throughput, use cases for databases and VMs |
| [Blob Storage](foundations/storage/blob-storage.md) | Binary large object storage for large payloads serving media and documents | Chunked upload, streaming reads, CDN integration |
| [Distributed File Systems](foundations/storage/distributed-file-systems.md) | Large-scale file storage optimized for sequential reads across many machines | Chunk servers, master metadata server, replication, rack awareness |
| [Data Replication Across Data Centers](foundations/storage/cross-dc-replication.md) | Multi-region data durability and disaster recovery | Synchronous vs asynchronous replication, RPO, RTO, active-active vs active-passive |

---

### 12. Distributed Systems Internals

> The foundational algorithms and data structures that make distributed systems work correctly.

| Topic | Description | Key Sub-concepts |
|---|---|---|
| [Distributed Consensus (Raft / Paxos)](foundations/distributed/consensus.md) | Achieving agreement on a value across nodes despite failures | Leader term, log replication, quorum commit, split vote |
| [Leader Election](foundations/distributed/leader-election.md) | Dynamically selecting a single coordinator node in a cluster | Bully algorithm, Raft election, fencing tokens, split-brain prevention |
| [Distributed Coordination Services](foundations/distributed/coordination-services.md) | Dedicated systems for cluster coordination, configuration, and service discovery | Zookeeper (ZAB protocol), etcd (Raft), Consul, ephemeral nodes, watches |
| [Distributed Locks](foundations/distributed/distributed-locks.md) | Mutual exclusion across processes running on different machines | Lease-based locks, Redlock, fencing tokens, lock expiry and renewal |
| [Distributed Transactions](foundations/distributed/distributed-transactions.md) | Atomicity across services or data stores | Two-Phase Commit (2PC), coordinator failure, Saga as a 2PC alternative |
| [Vector Clocks & Lamport Timestamps](foundations/distributed/vector-clocks.md) | Logical clocks for tracking causality and event ordering in distributed systems | Happens-before relation, conflict detection, last-write-wins |
| [Gossip Protocol](foundations/distributed/gossip-protocol.md) | Peer-to-peer state propagation without a central coordinator | Epidemic dissemination, failure detection, eventual consistency convergence |
| [Bloom Filters](foundations/distributed/bloom-filters.md) | Probabilistic data structure answering "definitely not in set / possibly in set" | False positive rate, size vs accuracy trade-off, use in caching and routing |
| [Merkle Trees](foundations/distributed/merkle-trees.md) | Hash trees enabling efficient comparison and verification of data across nodes | Anti-entropy repair, integrity verification, use in databases and blockchain |
| [Quorum](foundations/distributed/quorum.md) | Read and write quorum strategies for tuning consistency vs availability | R + W > N rule, quorum reads/writes, consistency levels |

---

### 13. Security

> Protecting systems, data, and users from unauthorized access and malicious behavior.

| Topic | Description | Key Sub-concepts |
|---|---|---|
| [Authentication & Authorization](foundations/security/authn-authz.md) | Verifying identity and enforcing access control | OAuth 2.0, JWT, session tokens, role-based access control (RBAC) |
| [Encryption in Transit & at Rest](foundations/security/encryption.md) | Protecting data as it moves and while stored | TLS/mTLS, certificate management, database encryption, key management |
| [DDoS Protection](foundations/security/ddos.md) | Mitigating distributed denial-of-service attacks | Rate limiting, IP filtering, anycast routing, challenge-response (CAPTCHA) |
| [API Security](foundations/security/api-security.md) | Securing service endpoints from misuse and exploitation | API keys, HMAC signing, input validation, whitelisting, throttling |
| [Principle of Least Privilege](foundations/security/least-privilege.md) | Every component should have the minimal access rights needed to do its job | Service-to-service auth, secrets management, network segmentation |

---

### 14. Observability

> Understanding the internal state of a system through its external outputs.

| Topic | Description | Key Sub-concepts |
|---|---|---|
| [Logging](foundations/observability/logging.md) | Centralized collection and querying of structured log data | Structured vs unstructured logs, log levels, sampling, retention |
| [Metrics & Alerting](foundations/observability/metrics-alerting.md) | Measuring system health and triggering notifications on anomalies | Counters, gauges, histograms, percentiles, alerting fatigue |
| [Distributed Tracing](foundations/observability/distributed-tracing.md) | End-to-end visibility of a request as it flows through multiple services | Trace ID / Span ID, correlation IDs, sampling strategies |
| [Dashboards & Visualization](foundations/observability/dashboards.md) | Aggregating metrics into real-time operational visibility | RED method (Rate, Errors, Duration), USE method, golden signals |
| [Chaos Engineering](foundations/observability/chaos-engineering.md) | Deliberately injecting failures to discover weaknesses before they occur in production | Game days, blast radius, steady-state hypothesis, controlled experiments |

---

## Section 2 — Case Studies

> End-to-end design problems that combine the foundational concepts above into realistic systems. Each case study highlights the specific design challenges that make that system interesting.

---

### A. Social Media & Feeds

| Case Study | Core Challenge | Key Design Considerations |
|---|---|---|
| [Design Instagram](case-studies/social/instagram.md) | Photo sharing with a global social graph and ranked feed | Chunked media upload, async transcoding, hybrid fan-out (push/pull), Redis feed cache |
| [Design Twitter Feed](case-studies/social/twitter-feed.md) | Low-latency personalized timeline for 300M users | Snowflake IDs, Cassandra tweet store, fan-out on write with celebrity hybrid, Redis List timeline |
| [Design Facebook News Feed](case-studies/social/facebook-news-feed.md) | Aggregate and rank content from a user's entire social graph in real time | Multi-stage candidate retrieval + ML ranking, TAO social graph, EdgeRank, seen-post dedup |
| [Design TikTok](case-studies/social/tiktok.md) | High-engagement personalized short-video feed with viral growth | Interest graph, HLS adaptive streaming, two-stage ANN+ranker recommendation, cold start |

---

### B. Messaging & Real-time Communication

| Case Study | Core Challenge | Key Design Considerations |
|---|---|---|
| [Design WhatsApp](case-studies/messaging/whatsapp.md) | Deliver messages reliably at 500M DAU with end-to-end encryption | WebSocket connection management, Redis presence map, Cassandra messages, Signal Protocol E2EE |
| [Design Slack](case-studies/messaging/slack.md) | Team messaging with workspace isolation, search, and presence | Workspace multi-tenancy, Cassandra messages, unread cursor, Elasticsearch with ACL-scoped search |

---

### C. Video & Media Streaming

| Case Study | Core Challenge | Key Design Considerations |
|---|---|---|
| [Design YouTube](case-studies/streaming/youtube.md) | Upload, transcode, store, and stream video to billions of users globally | Parallel segment transcoding, HLS adaptive bitrate, CDN pre-warming, Redis view count buffer |
| [Design Netflix](case-studies/streaming/netflix.md) | Deliver high-quality video globally with 99.99% availability | Open Connect own CDN, per-scene encoding, DRM triple-stack, proactive content pre-positioning |
| [Design Spotify](case-studies/streaming/spotify.md) | Stream audio with offline support and personalized discovery | OGG/AAC encoding, gapless pre-fetch, offline DRM license TTL, Discover Weekly batch generation |

---

### D. Location & Transportation

| Case Study | Core Challenge | Key Design Considerations |
|---|---|---|
| [Design Uber / Lyft](case-studies/location/uber-lyft.md) | Match riders to nearby drivers in real time with dynamic pricing | Redis GEOADD, 500K location updates/sec, 8s TTL, sequential dispatch, H3 hex surge pricing |
| [Design Google Maps](case-studies/location/google-maps.md) | Provide routing with real-time traffic across the entire world | Vector tiles, Contraction Hierarchies routing, HMM map-matching, crowd-sourced traffic |

---

### E. Search & Discovery

| Case Study | Core Challenge | Key Design Considerations |
|---|---|---|
| [Design a Web Search Engine](case-studies/search/web-search-engine.md) | Crawl the web, index billions of pages, return relevant results in milliseconds | Polite crawler with Bloom filter, inverted index, TF-IDF + PageRank + neural ranking, DAAT query |
| [Design Elasticsearch at Scale](case-studies/search/elasticsearch-at-scale.md) | Distributed full-text search over billions of documents at sub-100ms latency | Lucene segments, scatter-gather, BM25, erasure coding, HNSW vector search |
| [Design Typeahead / Autocomplete](case-studies/search/typeahead-autocomplete.md) | Return ranked prefix-match suggestions with sub-50ms latency | Top-K Trie, Redis ZRANGEBYLEX, frequency+recency scoring, client debouncing, personalization |

---

### F. Storage & File Sync

| Case Study | Core Challenge | Key Design Considerations |
|---|---|---|
| [Design Dropbox / Google Drive](case-studies/storage/dropbox-google-drive.md) | Sync files across devices with delta sync, conflict resolution, and versioning | 4 MB chunking, content-hash dedup, resumable upload, conflict copy, WebSocket sync push |
| [Design S3-like Object Storage](case-studies/storage/s3-object-store.md) | Store and retrieve arbitrary blobs durably at exabyte scale | Erasure coding (12+4), consistent hashing, multipart upload, presigned URLs, strong consistency |

---

### G. E-commerce & Payments

| Case Study | Core Challenge | Key Design Considerations |
|---|---|---|
| [Design Amazon Checkout](case-studies/ecommerce/amazon-checkout.md) | Process purchases consistently across inventory, payment, and order services | Saga pattern with compensation, idempotency keys, payment auth-then-capture, inventory reservation |
| [Design a Flash Sale System](case-studies/ecommerce/flash-sale.md) | Handle 1M concurrent purchases for 10K items without overselling | Redis Lua atomic DECR, request queue + async workers, bot protection, reservation expiry |

---

### H. Infrastructure & Utilities

| Case Study | Core Challenge | Key Design Considerations |
|---|---|---|
| [Design a URL Shortener](case-studies/infrastructure/url-shortener.md) | Generate short unique codes and redirect at massive scale with analytics | Counter vs hash-based codes, 301 vs 302 redirect, Redis cache, range-based distributed counter |
| [Design a CDN](case-studies/infrastructure/cdn-design.md) | Distribute and cache content at global edge locations | Anycast routing, cache key design, request collapsing, stale-while-revalidate, instant purge |
| [Design a Web Crawler](case-studies/infrastructure/web-crawler.md) | Crawl billions of web pages efficiently without overloading servers | Politeness (robots.txt), Bloom filter URL dedup, SimHash near-dup detection, per-domain rate limiting |
| [Design a Distributed Job Scheduler](case-studies/infrastructure/distributed-job-scheduler.md) | Schedule and execute millions of jobs exactly-once across a worker fleet | Time-indexed DB polling, transactional outbox, leader election, heartbeat reaper, hot-second mitigation |

---

### I. Analytics & Data Systems

| Case Study | Core Challenge | Key Design Considerations |
|---|---|---|
| [Design a Metrics Analytics Pipeline](case-studies/analytics/metrics-analytics-pipeline.md) | Ingest 1M events/sec, serve real-time dashboards and historical queries | Kafka → Flink → ClickHouse; Lambda vs Kappa; late event handling; materialized views |
| [Design an Ad Click Tracker](case-studies/analytics/ad-click-tracker.md) | Count ad clicks exactly-once at 11K/sec with fraud detection | Signed click tokens, fire-and-forget Kafka, Bloom filter dedup, streaming fraud signals, batch billing |

---

### J. Gaming & Leaderboards

| Case Study | Core Challenge | Key Design Considerations |
|---|---|---|
| [Design a Leaderboard System](case-studies/gaming/leaderboard.md) | Maintain and serve real-time ranking for 100M players at sub-10ms | Redis Sorted Set ZADD/ZREVRANK, Top-K cached at O(log N), tie-breaking, daily rotation |
| [Design a Multiplayer Game Backend](case-studies/gaming/multiplayer-game.md) | Synchronize shared game state across players with minimal perceived latency | UDP vs TCP, authoritative server, client-side prediction, lag compensation, interest management |

---

## Contributing

Each topic in this repository will have its own dedicated Markdown file organized under the following folder structure:

```
foundations/
  core-concepts/
  reliability/
  networking/
  traffic/
  databases/
  nosql/
  caching/
  scalability/
  messaging/
  microservices/
  storage/
  distributed/
  security/
  observability/

case-studies/
  social/
  messaging/
  streaming/
  location/
  search/
  storage/
  ecommerce/
  infrastructure/
  analytics/
  gaming/
```

When adding a new topic file, the structure should include:
- **What it is** — a clear definition
- **How it works** — the mechanics, with diagrams where helpful
- **Key trade-offs** — what you gain and what you give up
- **When to use it** — the problems it solves and the conditions under which it applies
- **Common interview / design pitfalls** — mistakes to avoid

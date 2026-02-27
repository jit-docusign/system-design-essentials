# Apache Kafka

## What it is

**Apache Kafka** is a distributed, replicated **commit log** (or event streaming platform) designed for high-throughput, fault-tolerant ingestion and delivery of event streams. Unlike traditional message queues that delete messages upon consumption, Kafka **retains messages for a configurable retention period** (e.g., 7 days), allowing multiple consumers to independently read the same data at their own pace.

Kafka was originally built at LinkedIn and is now used at scale for activity tracking, metrics pipelines, log aggregation, and event sourcing.

---

## How it works

### Core Architecture

```
                           Kafka Cluster
                    ┌──────────────────────────┐
                    │  Broker 1    Broker 2    │
                    │  ┌────────┐  ┌────────┐  │
Producers           │  │ Topic A│  │ Topic A│  │  Consumer Groups
  │  produce(topic) │  │ Part 0 │  │ Part 1 │  │
  ├────────────────►│  │[leader]│  │[leader]│  │◄──── Group A (analytics)
  │                 │  └────────┘  └────────┘  │◄──── Group B (notifications)
  │                 │  Broker 3               │
  │                 │  ┌────────┐             │
  │                 │  │ Topic A│             │
  │                 │  │ Part 2 │             │
  │                 │  │[leader]│             │
  │                 │  └────────┘             │
                    └──────────────────────────┘
```

### Topics, Partitions, Offsets

**Topic**: a named, append-only log — like a database table. e.g., `orders`, `user-events`, `payment-results`.

**Partition**: each topic is split into N partitions. Partitioning enables parallel processing and horizontal scaling.

```
Topic "orders"  (3 partitions)

Partition 0: [ msg0 | msg1 | msg4 | msg7 | ... ]
                 ↑     ↑     ↑     ↑
              offset 0  1     2     3

Partition 1: [ msg2 | msg5 | msg8 | ... ]
Partition 2: [ msg3 | msg6 | msg9 | ... ]
```

Key rule: **messages with the same key always go to the same partition** (round-robin if no key). Use meaningful keys to group related events:
- `order_id` → all events for one order land in the same partition → total ordering per order.
- No key → events spread across partitions → maximum throughput but no ordering.

**Offset**: the position of a message within a partition. Consumers track their offset and commit it. A consumer restart resumes from its last committed offset.

### Consumer Groups

A **consumer group** is a set of consumers that share the work of consuming a topic. Each partition is assigned to exactly one consumer in the group.

```
Topic "orders" — 4 partitions

Consumer Group A (2 consumers):
  Consumer A1 → reads Partition 0 + Partition 1
  Consumer A2 → reads Partition 2 + Partition 3

Consumer Group B (4 consumers):
  Consumer B1 → reads Partition 0
  Consumer B2 → reads Partition 1
  Consumer B3 → reads Partition 2
  Consumer B4 → reads Partition 3
```

Groups A and B are **independent** — each gets all messages. This enables fan-out: analytics and notifications both read the full `orders` topic independently.

**Rebalancing** occurs when a consumer joins or leaves the group — partitions are redistributed. Modern Kafka uses **cooperative incremental rebalancing** to minimize disruption.

Scaling rule: **you cannot have more consumers in a group than partitions in the topic** — extra consumers sit idle. Increase partitions to scale consumers.

### Replication

Each partition has one **leader replica** and N-1 **follower replicas** on other brokers. Producers write to the leader; followers replicate.

```
Partition 0:  Broker 1 (leader) ←── writes
              Broker 2 (follower) ←── replicates from leader
              Broker 3 (follower) ←── replicates from leader
```

**ISR (In-Sync Replicas)**: the set of replicas caught up to the leader. `acks=all` means the leader waits for all ISR members to acknowledge before responding to the producer.

```
Producer config:
  acks=0    → fire-and-forget, no durability guarantee
  acks=1    → leader acknowledges, fastest with some durability
  acks=all  → all ISR acknowledge, strongest durability
```

### Log Compaction

By default, Kafka retains messages by **time** (e.g., 7 days) or **size**. With **log compaction**, Kafka retains only the **latest message per key**, indefinitely. Perfect for maintaining current state (e.g., a changelog topic for a user's profile).

```
Before compaction:        After compaction:
Key=user1 → value=A       Key=user1 → value=C  (latest)
Key=user2 → value=X       Key=user2 → value=X
Key=user1 → value=B       (user1-A and user1-B removed)
Key=user1 → value=C
```

### Producer and Consumer Code

```python
from kafka import KafkaProducer, KafkaConsumer
import json

# Producer
producer = KafkaProducer(
    bootstrap_servers=['kafka:9092'],
    value_serializer=lambda v: json.dumps(v).encode('utf-8'),
    acks='all',
    retries=3,
    enable_idempotence=True  # exactly-once producer semantics
)

producer.send('orders', key=b'order-123', value={
    'event_type': 'order.placed',
    'order_id': 'order-123',
    'total': 59.98
})
producer.flush()

# Consumer
consumer = KafkaConsumer(
    'orders',
    bootstrap_servers=['kafka:9092'],
    group_id='analytics-service',
    auto_offset_reset='earliest',   # start from beginning if no committed offset
    enable_auto_commit=False,        # manual commit for at-least-once
    value_deserializer=lambda m: json.loads(m.decode('utf-8'))
)

for msg in consumer:
    process_event(msg.value)
    consumer.commit()  # commit after successful processing
```

### Kafka Streams vs Producers/Consumers

**Kafka Producers/Consumers**: low-level primitives — you manage offset, error handling, state.

**Kafka Streams**: Java library for stateful stream processing on top of Kafka. Handles joins, aggregations, windowing.

```java
// Kafka Streams: count orders per user in 1-minute tumbling windows
KStream<String, Order> orders = builder.stream("orders");
orders
    .groupByKey()
    .windowedBy(TimeWindows.of(Duration.ofMinutes(1)))
    .count()
    .toStream()
    .to("order-counts-per-minute");
```

---

## Key Trade-offs

| Advantage | Disadvantage |
|---|---|
| Extremely high throughput (millions of msg/sec) | Operationally complex to run and tune |
| Durable replay — consumers can re-read old messages | High latency possible if not tuned (default batching adds ms) |
| Multiple independent consumer groups | Partition count hard to change post-creation (requires migration) |
| Log compaction for event sourcing/CDC | Kafka is a poor traditional message queue — no per-message TTL |
| Built-in replication and fault tolerance | Ordering only within a partition |
| Horizontal scaling via partition count | Zookeeper/KRaft meta-management overhead |

---

## When to use

- **Activity tracking at scale**: page views, click streams, sensor readings — millions of events/sec.
- **Event sourcing backbone**: durable commit log for aggregate events.
- **Change Data Capture (CDC)**: database row changes streamed to downstream systems via Debezium.
- **Microservices async communication**: services emit and consume domain events.
- **Replayable pipelines**: re-process all events from time T to rebuild a projection.
- **Log aggregation**: aggregate app logs from many instances for centralized processing.

Use a simpler queue (SQS, RabbitMQ) when:
- You don't need replay.
- Consumer groups are not required.
- You need per-message priority or complex routing.

---

## Common Pitfalls

- **Too few partitions**: you can't have more parallel consumers (in a group) than partitions. Under-partition early and you're bottlenecked. Rule of thumb: start with max(target_consumer_count, target_producer_throughput / single_partition_throughput).

- **Not committing offsets correctly**: auto-commit can lead to data loss (commit before processing completes) or duplicate processing (process, then commit fails on crash). Use manual commit after successful processing.

- **Using Kafka for RPC**: Kafka is not designed for synchronous request/reply. Adding a reply topic and correlation ID works but is complex. Use gRPC or REST for request/reply.

- **Ignoring consumer lag**: rising consumer lag = consumers can't keep up. Monitor `kafka.consumer_lag` and alert. Scale consumers or optimize processing before the lag becomes a problem.

- **Large message payloads**: Kafka is optimized for many small messages. Avoid storing blobs in Kafka — store large payloads in object storage (S3/GCS) and put just the reference in the Kafka message.

- **Hot partitions**: a bad partition key (e.g., timestamp, or a key where one value dominates) causes a hot partition receiving all or most traffic. Choose high-cardinality, uniformly distributed keys.

# Design Slack

## Problem Statement

Design a workplace messaging platform supporting persistent threaded channels, direct messages, file sharing, search, and real-time delivery across web, desktop, and mobile clients.

**Scale requirements (Slack)**:
- 20 million DAU, 750 million messages per day
- 150,000 organizations (workspaces)
- Channels with up to 10,000 members
- < 200ms message delivery
- Must support message history / search going back years

---

## What Makes Slack Different from WhatsApp

| Dimension | Slack | WhatsApp |
|---|---|---|
| Primary context | Teams / workspaces (organization-centric) | Personal contacts (individual-centric) |
| History | Persistent, searchable forever | 30-day cloud backup |
| Channels | Group channels are the primary model | 1:1 is primary; groups are secondary |
| Threading | First-class threads in channels | Replies as separate conversations |
| Integrations | Bots, webhooks, app directory | No third-party integrations |
| Encryption | Transport-layer (server can read) | End-to-end encrypted |

---

## High-Level Architecture

```
Client (Web/Desktop/Mobile)
         │
         │ WebSocket (persistent)
         ▼
┌─────────────────────────────────────────────────────────────┐
│                    Gateway Servers                          │
│  (WebSocket handler: 1 persistent connection per client)    │
│           + Presence Service (Redis)                        │
└──────┬──────────────────────┬──────────────────────────────┘
       │                      │
       ▼                      ▼
Messaging Service         Channel Service
       │                      │
       ▼                      ▼
Message Store           Channel/User DB
(Cassandra)              (Sharded MySQL)
       │
       ▼
Search Indexer
(Elasticsearch)
```

---

## How it works

### Workspace Isolation (Multi-Tenancy)

Slack organizes all data by **workspace (team_id)**:

```sql
-- All queries start with workspace_id (tenant isolation)
-- MySQL sharded by workspace_id

CREATE TABLE channels (
    workspace_id UUID,
    channel_id   UUID,
    name         VARCHAR(80),
    is_private   BOOLEAN,
    member_count INT,
    created_at   TIMESTAMPTZ,
    PRIMARY KEY (workspace_id, channel_id)
);

CREATE TABLE channel_members (
    workspace_id UUID,
    channel_id   UUID,
    user_id      UUID,
    joined_at    TIMESTAMPTZ,
    last_read    TIMESTAMPTZ,  -- for unread count calculation
    PRIMARY KEY (workspace_id, channel_id, user_id)
);
```

Large enterprise workspaces get dedicated MySQL shards. Small workspaces share shards.

### Message Delivery Flow

```
User A sends message to #general:
  1. Client sends via WebSocket to Gateway Server
     {workspace_id, channel_id, sender_id, content, client_msg_id}

  2. Gateway routes to Messaging Service

  3. Messaging Service:
     a. Generate message_id (Snowflake)
     b. Persist to Cassandra message store
     c. Update channel's latest_activity timestamp (MySQL)
     d. Publish to internal fanout bus:
        topic: "workspace:{workspace_id}:channel:{channel_id}"
        payload: {message_id, sender_id, content, timestamp}
     e. Return ACK to sender with server-assigned message_id

  4. Fanout Service:
     a. Read #general member list
     b. For each member:
        - If online: route message to their Gateway Server connection
        - If offline: skip (they'll pull history on reconnect)

  5. Each active member's client receives the message via WebSocket

  6. Notification Service:
     a. For each member not currently viewing this channel:
        - Check notification preferences (mention, keywords, etc.)
        - Send push notification (APNs/FCM) if preferences match
```

### Message Store (Cassandra)

```sql
-- Cassandra: messages ordered by time within a channel
CREATE TABLE messages (
    workspace_id UUID,
    channel_id   UUID,
    message_id   BIGINT,   -- Snowflake (time-sortable)
    sender_id    UUID,
    content      TEXT,
    content_type VARCHAR(20),  -- 'message', 'file', 'bot_message'
    thread_ts    BIGINT,        -- NULL for top-level; parent message_id for thread replies
    edited_at    TIMESTAMP,
    deleted_at   TIMESTAMP,
    reactions    MAP<TEXT, SET<UUID>>,  -- {"👍": {user1, user2}, "rocket": {user3}}
    PRIMARY KEY ((workspace_id, channel_id), message_id)
) WITH CLUSTERING ORDER BY (message_id DESC);

-- Load last 50 messages:
SELECT * FROM messages WHERE workspace_id = ? AND channel_id = ? LIMIT 50;

-- Load messages before a cursor (pagination):
SELECT * FROM messages WHERE workspace_id = ? AND channel_id = ? 
  AND message_id < ? LIMIT 50;
```

### Threads

Threads in Slack are replies to a specific message. They're stored as regular messages with a `thread_ts` pointer to the parent:

```
#general channel view:
  msg-100: "Has anyone seen the Q3 report?" (has 3 replies)
  msg-200: "Weekly standup link: zoom.com/..."
  msg-300: "AWS is down again"

Thread view for msg-100:
  msg-100: "Has anyone seen the Q3 report?"  (parent message)
  msg-101: "Yes! It's in the shared drive"       (thread_ts = 100)
  msg-102: "Link: drive.google.com/..."           (thread_ts = 100)
  msg-103: "Thanks!"                              (thread_ts = 100)
```

```sql
-- Fetch thread:
SELECT * FROM messages WHERE workspace_id = ? AND channel_id = ?
  AND thread_ts = ?
  ORDER BY message_id ASC;

-- Fetch channel messages (top-level only, with reply count):
SELECT * FROM messages WHERE workspace_id = ? AND channel_id = ?
  AND thread_ts IS NULL
  LIMIT 50;
```

### Presence and Typing Indicators

```python
class PresenceService:
    """Redis-backed presence tracking"""
    
    async def user_online(self, workspace_id: str, user_id: str, server_id: str):
        # Set presence with TTL (auto-expire if WebSocket drops)
        await redis.setex(f'presence:{workspace_id}:{user_id}', 90, server_id)
    
    async def user_offline(self, workspace_id: str, user_id: str):
        await redis.delete(f'presence:{workspace_id}:{user_id}')
    
    async def get_online_members(self, workspace_id: str, user_ids: list[str]) -> dict:
        keys = [f'presence:{workspace_id}:{uid}' for uid in user_ids]
        values = await redis.mget(keys)
        return {uid: val is not None for uid, val in zip(user_ids, values)}

# Typing indicator: ephemeral, not persisted
class TypingIndicator:
    async def user_typing(self, workspace_id: str, channel_id: str, user_id: str):
        # Publish typing event — TTL 5s (no ACK needed, ephemeral)
        await redis.setex(f'typing:{workspace_id}:{channel_id}:{user_id}', 5, 1)
        await fanout.publish(f'workspace:{workspace_id}:channel:{channel_id}', {
            'type': 'typing_start',
            'user_id': user_id
        })
```

### Unread Counts

```python
# Track unread messages per user per channel
async def get_unread_count(user_id: str, workspace_id: str, channel_id: str) -> int:
    # last_read is the timestamp of the last message the user has seen
    last_read_ts = await db.fetchval(
        "SELECT last_read FROM channel_members WHERE user_id = ? AND channel_id = ?",
        user_id, channel_id
    )
    
    # Count messages after last_read
    return await cassandra.fetchval(
        "SELECT COUNT(*) FROM messages WHERE workspace_id = ? AND channel_id = ? "
        "AND message_id > ? AND thread_ts IS NULL",
        workspace_id, channel_id, timestamp_to_snowflake(last_read_ts)
    )

# Mark channel as read (update last_read timestamp)
async def mark_channel_read(user_id: str, channel_id: str) -> None:
    last_message_id = await get_latest_message_id(channel_id)
    await db.execute(
        "UPDATE channel_members SET last_read = ? WHERE user_id = ? AND channel_id = ?",
        snowflake_to_timestamp(last_message_id), user_id, channel_id
    )
```

### Search

Full-text search of message history is a critical Slack feature:

```
Message posted → Kafka topic 'messaging-events'
                      │
                 Search Indexer Worker
                      │
                 Elasticsearch Index
                 {workspace_id, channel_id, sender_id, ts, content}

User searches "AWS deployment pipeline":
  → Elasticsearch: bool query with user access filter
    {must: {match: "AWS deployment pipeline"}, filter: {terms: {channel_id: [accessible_channels]}}}
  → Ranked by relevance + recency
  → Return message snippets with highlights
```

**Access control for search**: users can only search channels they're members of. The search query must filter by `channel_id` in the set of channels the user has access to — computed at search time, not stored in the index.

### File Storage

```
File upload flow:
  1. Client requests presigned S3 PUT URL from Slack API
  2. Client uploads directly to S3 (bypasses Slack servers for large files)
  3. Client calls /files.complete → Slack creates file metadata record
  4. A "file_share" message is posted to the channel with file metadata
  5. Recipients: click "Download" → redirect to presigned S3 GET URL (time-limited)

Media previews (image thumbnails, video previews):
  → Generated async by media processing workers after upload
  → Stored in S3, served via Slack CDN
```

### Real-Time Fan-out at Channel Scale

For a channel with 10,000 members, delivering a message requires notifying up to 10,000 possibly-online users:

```
Channel has 10,000 members; 500 currently online across 50 gateway servers.

Fanout Service:
  1. Get channel member online status (batch Redis MGET)
  2. Group online members by their gateway server:
     {server-1: [user1, user2, ...], server-2: [user5, user6, ...], ...}
  3. For each gateway server, send one message with the list of recipients
     → Server delivers individually to each WebSocket

Result: 50 internal messages (1 per server) instead of 500 individual deliveries
```

---

## Key Design Decisions

| Decision | Choice | Reasoning |
|---|---|---|
| Message storage | Cassandra | Time-ordered per channel; write-heavy workload |
| Multi-tenancy | Sharded MySQL by workspace_id | Tenant isolation; large enterprises on dedicated shards |
| Search | Elasticsearch | Full-text search with access-control filtering |
| Connection protocol | WebSocket | Real-time bidirectional messaging |
| Fan-out | Server-grouped batch delivery | Reduces internal messages for large channels |
| File upload | Presigned S3 PUT (direct) | Don't route large files through API servers |

---

## Scaling Challenges

| Challenge | Solution |
|---|---|
| 10,000-member channel fan-out | Server-grouped delivery via internal message bus |
| Persistent message history | Cassandra time-series schema; S3 for old data archiving |
| Full-text search with access control | Elasticsearch with per-query channel membership filter |
| Multi-tenant isolation | MySQL sharding by workspace_id |
| Unread counts at scale | Store last_read cursor; COUNT messages > cursor in Cassandra |

---

## Hard-Learned Engineering Lessons

- **Not modeling the workspace boundary**: all Slack data is workspace-scoped. Omitting this leads to cross-tenant data leaks in the design.
- **Using a relational DB for messages**: message volume (750M/day) and time-series access pattern (last N messages in a channel) are not a good fit for a single relational DB.
- **Ignoring threading**: Slack threads are important enough to design explicitly. Without thread storage, the schema is wrong.
- **Searching without access control**: search results must respect channel membership. Users should not see messages from private channels they're not in — this must be enforced at the Elasticsearch query level.

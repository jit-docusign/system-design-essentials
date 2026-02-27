# Design WhatsApp / Messenger

## Problem Statement

Design a real-time, end-to-end encrypted messaging platform supporting 1:1 chats, group chats, media sharing, and delivery receipts (sent / delivered / read).

**Scale requirements (WhatsApp)**:
- 2 billion users, 500 million DAU
- 100 billion messages sent per day (~1.2 million messages/second)
- Groups up to 1,024 members
- Maximum media file size: 100 MB
- Message delivery in < 100ms when both users are online

---

## Core Features

1. **1:1 messaging** — text, emoji, voice messages
2. **Group chats** — up to 1,024 members
3. **Media sharing** — photos, videos, documents
4. **Delivery status** — ✓ Sent, ✓✓ Delivered, ✓✓ Read (blue ticks)
5. **End-to-end encryption** (E2EE)
6. **Offline delivery** — messages queued when recipient is offline

---

## High-Level Architecture

```
Client A                  Servers                  Client B
   │                         │                         │
   │── connect (WebSocket) ──►│◄── connect (WebSocket) ─│
   │                         │                         │
   │── send message ─────────►│── deliver message ──────►│
   │◄── ACK (sent ✓) ─────── │                         │
   │                         │◄── delivered ACK ────── │
   │◄── delivered ✓✓ ──────── │                         │
   │                         │                         │
   │                     (if B offline)                │
   │                    [store in queue]                │
   │                    [deliver when B reconnects]     │
```

---

## How it works

### Connection Layer — WebSockets

All clients maintain a persistent WebSocket connection to a message router server:

```
Mobile Client
     │
     │ TCP WebSocket (persistent, long-lived)
     │
     ▼
┌──────────────────────────────────────────────────────┐
│  Connection Service Layer                             │
│                                                      │
│  Server 1: 10,000 WebSocket connections              │
│  Server 2: 10,000 WebSocket connections              │
│  ...                                                 │
│  (50,000 servers × 10,000 connections = 500M DAU)    │
└──────────────────────────────────────────────────────┘
     │
     ▼
  Presence Service (Redis)
  connection_server_map: {user_id → server_id}
```

Why WebSocket?
- Bidirectional: server can push messages to client without client polling.
- Low overhead: no HTTP header per message.
- Persistent: no reconnect latency per message.

Each message server maintains an in-memory map of `{user_id: websocket_connection}` for users currently connected to that server.

### Message Flow — Online Recipient

```
1. Client A sends message to B
   {msg_id, sender_id, recipient_id, content_encrypted, timestamp}

2. Message Router Server (MSS) receives the message
   a. Persist message to Message Store (Cassandra) → status: sent
   b. Lookup: "Which server is user B connected to?"
      → Redis: connection_server_map[B.user_id] = "server-42"
   c. Forward to Server-42

3. Server-42 delivers message to Client B via WebSocket

4. Client B sends delivery ACK → Server-42 → Message Router → Client A
   → Update message status: delivered (✓✓)

5. Client B opens conversation (reads) → sends read receipt
   → Update message status: read (blue ✓✓)
```

```python
# Simplified message routing
class MessageRouter:
    def __init__(self, presence_redis, message_store, mq):
        self.presence = presence_redis
        self.store = message_store
        self.mq = mq  # internal message bus between servers
    
    async def send_message(self, message: Message) -> str:
        # 1. Generate message ID (Snowflake)
        message.id = generate_snowflake_id()
        
        # 2. Persist to Cassandra
        await self.store.save(message)
        
        # 3. ACK sender (message is persisted = sent ✓)
        await self.ack_sender(message.sender_id, message.id, status='sent')
        
        # 4. Route to recipient
        server_id = await self.presence.get(f'conn_server:{message.recipient_id}')
        
        if server_id:
            # Online: route to the server holding recipient's WebSocket
            await self.mq.publish(f'deliver:{server_id}', message)
        else:
            # Offline: message already persisted, will be delivered on reconnect
            await self.store.update_status(message.id, 'pending')
        
        return message.id

class ConnectionServer:
    connections: dict[str, WebSocket] = {}
    
    async def on_connect(self, user_id: str, ws: WebSocket) -> None:
        self.connections[user_id] = ws
        
        # Register in presence service
        await self.presence.setex(f'conn_server:{user_id}', 3600, self.server_id)
        
        # Deliver any pending messages from Cassandra
        pending = await message_store.get_pending(user_id)
        for msg in pending:
            await ws.send(msg)
            await message_store.update_status(msg.id, 'delivered')
    
    async def on_disconnect(self, user_id: str) -> None:
        del self.connections[user_id]
        await self.presence.delete(f'conn_server:{user_id}')
    
    async def deliver(self, message: Message) -> None:
        ws = self.connections.get(message.recipient_id)
        if ws:
            await ws.send(message)
            await message_store.update_status(message.id, 'delivered')
```

### Message Storage — Cassandra

Messages are stored per conversation (chat_id), clustered by time-ordered message IDs:

```sql
-- Chat messages table (Cassandra)
CREATE TABLE messages (
    chat_id      UUID,       -- 1:1 chat or group ID
    message_id   BIGINT,     -- Snowflake ID (time-sortable)
    sender_id    UUID,
    content      BLOB,       -- Encrypted content (E2EE: server stores ciphertext only)
    content_type VARCHAR(20), -- 'text', 'image', 'video', 'audio'
    media_key    BLOB,       -- Encrypted media key (for media messages)
    status       VARCHAR(10), -- 'sent', 'delivered', 'read'
    created_at   TIMESTAMP,
    PRIMARY KEY (chat_id, message_id)
) WITH CLUSTERING ORDER BY (message_id DESC);

-- Load last 50 messages in a chat:
SELECT * FROM messages WHERE chat_id = ? LIMIT 50;
```

Why Cassandra:
- Write-heavy (100B messages/day)
- Time-series access pattern (recent messages first)
- Linear horizontal scaling
- Cross-datacenter replication built in

### Group Messaging

Groups up to 1,024 members. Fan-out happens on the server:

```python
async def send_group_message(message: Message) -> None:
    # Persist once
    await message_store.save(message)
    
    # Fan-out to all group members
    members = await group_service.get_members(message.chat_id)
    
    online_members = []
    offline_members = []
    for member_id in members:
        if member_id == message.sender_id:
            continue
        server_id = await presence.get(f'conn_server:{member_id}')
        if server_id:
            online_members.append((member_id, server_id))
        else:
            offline_members.append(member_id)
    
    # Deliver to online members (route to their connection server)
    tasks = [mq.publish(f'deliver:{sid}', {**message, 'recipient_id': mid})
             for mid, sid in online_members]
    await asyncio.gather(*tasks)
    
    # Offline members: will pull on reconnect (already in message store)
```

For very large groups (future: 1M+ members like broadcast channels), fan-out at the server level becomes expensive. WhatsApp limits groups to 1,024 members partly to control fan-out cost.

### End-to-End Encryption (Signal Protocol)

WhatsApp uses the **Signal Protocol** for E2EE:

```
Key concepts:
  - Each device generates a long-term identity key pair (Ed25519)
  - Pre-keys: batch of one-time key pairs uploaded to server at registration
  - Session setup: ECDH key exchange to derive a shared secret (X3DH protocol)
  - Message encryption: AES-256-GCM with per-message ratcheting keys

What the server sees:
  - Sender ID, recipient ID, message size, timestamp
  - Ciphertext (cannot decrypt — no access to private keys)
  - Media: encrypted with a random key sent in the message body (also encrypted)

Server cannot read messages — truly E2EE.
```

```
Key Distribution Flow:
1. Alice installs WhatsApp → generates identity key + 100 pre-keys
   → Uploads public keys to WhatsApp key server

2. Alice sends first message to Bob:
   a. Fetch Bob's public identity key + one pre-key from server
   b. X3DH key agreement → shared secret
   c. Derive message key (Double Ratchet algorithm)
   d. Encrypt message with AES-256-GCM

3. Server receives ciphertext → stores, routes, cannot read

4. Bob decrypts using his private keys locally
```

### Media Messages

Media (photos, videos, documents) are too large for the message path:

```
1. Sender encrypts media locally:
   media_key = random 32 bytes
   encrypted_media = AES-256-GCM(media_key, raw_media)

2. Upload encrypted_media to WhatsApp media servers (via presigned URL)
   → Returns media_url

3. Send message containing: {media_url, encrypted media_key, SHA-256 hash}
   (media_key is itself encrypted with E2EE session key)

4. Recipient downloads encrypted_media from media_url
   Decrypts media_key from message
   Decrypts media using media_key
   Verifies SHA-256 hash

WhatsApp never has the media_key → cannot decrypt media files.
```

### Delivery Receipts (Status Ticks)

```
✓  (grey single tick)   = Message delivered to WhatsApp server
✓✓ (grey double tick)   = Message delivered to recipient's phone
✓✓ (blue double tick)   = Recipient has opened the conversation

Implementation:
  Recipient's app sends delivery ACK (after receiving)
  → Server updates message status, notifies sender (through WebSocket)

  Recipient opens conversation
  → App sends batch read receipt for all messages in conversation
  → Server updates all to 'read', notifies sender
```

---

## Key Design Decisions

| Decision | Choice | Reasoning |
|---|---|---|
| Connection protocol | WebSocket | Bidirectional push; low overhead vs HTTP polling |
| Message storage | Cassandra (sharded by chat_id) | High write throughput; time-ordered access |
| Presence tracking | Redis (TTL-based) | Fast lookup; auto-expires stale entries on disconnect |
| Encryption | Signal Protocol (E2EE) | Industry-standard; server cannot read messages |
| Group fan-out | Server-side to 1,024 max | Limit contains fan-out cost; consistent delivery |
| Media delivery | Direct to media CDN via presigned URL | Don't route large files through message servers |

---

## Scaling Challenges

| Challenge | Solution |
|---|---|
| 500M concurrent WebSocket connections | ~50K connection servers at 10K connections each |
| Message routing with server affinity | Redis presence map (user_id → server_id) |
| 100B messages/day write load | Cassandra distributed write at 1.2M/sec |
| Offline message delivery | Cassandra as queue; deliver on reconnect |
| Media at scale | Separate media servers + CDN; E2EE in transit |

---

## Hard-Learned Engineering Lessons

- **Using HTTP polling or long-polling**: WebSocket is the right choice for real-time bidirectional messaging. Polling creates unnecessary latency and server load.
- **Storing messages in a relational database**: 100B messages/day at 1.2M writes/sec will not scale on standard MySQL. Cassandra with a time-series schema is the right fit.
- **Ignoring offline delivery**: a very common omission. Messages must be durably queued when the recipient is offline and delivered reliably on reconnect.
- **Single server routing assumption**: in a distributed system, message server A doesn't know which WebSocket connection server holds user B's connection. The presence service (Redis) solves this lookup problem.

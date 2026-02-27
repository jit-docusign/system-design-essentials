# Design a Real-Time Multiplayer Game Backend

## Problem Statement

Design the backend for a real-time multiplayer online game — like a battle royale (Fortnite), MOBA (League of Legends), or first-person shooter. The game involves 10-100 players sharing a game world, with each player's position and actions needing to be synchronized in near-real-time (< 50ms latency for movement, ~100ms for game events).

**Scale requirements**:
- 50 million daily active players
- 10 million concurrent game sessions
- Each game session: 10-100 players
- Movement sync rate: 20 updates/second per player
- Acceptable action latency: < 100ms (responsive enough to feel real-time)
- Match duration: 10-30 minutes

---

## Key Challenges

1. **Low latency**: game state needs sub-100ms round-trip. HTTP request-response adds 100-200ms overhead → must use persistent connections and UDP.
2. **High throughput**: 10M sessions × 50 players × 20 updates/sec = 10 billion events/sec across the system.
3. **Authoritative server**: client can't be trusted to report its own position (cheating). Server must be the ground truth.
4. **Lag compensation**: network latency makes player actions appear to happen later than they did. Server must compensate to feel fair.

---

## Transport Protocol: TCP vs UDP vs WebSocket

```
TCP (reliable, ordered):
  + Guaranteed delivery, in-order
  - Head-of-line blocking: one dropped packet delays all subsequent packets
  - 200ms lag spike from a single packet loss → game freezes
  - NOT suitable for game state updates (movement positions)

UDP (unreliable, unordered):
  + No head-of-line blocking: dropped packet is just lost, others arrive fine
  + Lower latency (no retransmit wait)
  - No guaranteed delivery, no ordering
  - Suitable for: position updates (getting a slightly stale position is fine)
  - Not suitable for: critical events (death, score, items) — need ack

WebSocket (TCP-based, persistent):
  + Works in browsers, firewall-friendly (port 80/443)
  + Good enough for turn-based or moderately fast games
  - Head-of-line blocking problem (same as TCP)

QUIC / HTTP/3 (UDP-based with streams):
  + Per-stream reliability (no cross-stream head-of-line blocking)
  + Works over UDP
  + Increasingly supported by games (Fortnite uses DTLS over UDP)
  - More complex to implement than raw UDP

Typical game architecture:
  - Movement / position updates: raw UDP (custom protocol)
  - Critical events (join, death, items, chat): TCP or QUIC reliable stream
  - Browser-based games: WebSocket (only option available)
```

---

## Architecture

```
Game Client (player device)
    │
    │  UDP (game state, 20 Hz)
    │  TCP/WebSocket (matchmaking, chat, critical events)
    │
    ▼
┌────────────────────────────────────────────────────────────┐
│  Edge Network (anycast UDP endpoints near players)         │
│  - Route to nearest game server via latency-based routing  │
└─────────────────────────────┬──────────────────────────────┘
                              │
                   ┌──────────▼──────────┐
                   │  Game Server        │
                   │  (one per match)    │
                   │                     │
                   │  Authoritative game │
                   │  state: positions,  │
                   │  health, items      │
                   │                     │
                   │  Tick rate: 60 Hz   │
                   │  (16ms per tick)    │
                   └──────────┬──────────┘
                              │
          ┌───────────────────┼──────────────────┐
          │                   │                  │
          ▼                   ▼                  ▼
  ┌──────────────┐  ┌──────────────┐   ┌──────────────┐
  │ Match DB     │  │ Player State │   │ Replay Store │
  │ (outcome,    │  │ (Redis, TTL  │   │ (S3: full    │
  │  stats)      │  │  = match     │   │  match data  │
  │              │  │  duration)   │   │  for replays │
  └──────────────┘  └──────────────┘   └──────────────┘
```

---

## How It Works

### 1. Matchmaking

```python
class MatchmakingService:
    def __init__(self, redis_client):
        self.r = redis_client
    
    def enqueue_player(self, player_id: str, skill_rating: float, region: str):
        """
        Add player to matchmaking queue.
        Players are sorted by skill rating to find close matches.
        """
        queue_key = f"matchmaking:{region}:queue"
        self.r.zadd(queue_key, {player_id: skill_rating})
        self.r.expire(queue_key, 300)  # Remove from queue if no match in 5 min
    
    def find_match(self, region: str, players_needed: int = 10,
                   skill_tolerance: float = 100.0) -> list[str] | None:
        """
        Find N players with similar skill ratings.
        """
        queue_key = f"matchmaking:{region}:queue"
        
        # Get all players in queue
        queued = self.r.zrange(queue_key, 0, -1, withscores=True)
        if len(queued) < players_needed:
            return None  # Not enough players
        
        # Try to find a group where all players are within skill_tolerance of each other
        # Sliding window: sort by skill, find window of size N where max-min <= tolerance
        for i in range(len(queued) - players_needed + 1):
            window = queued[i:i + players_needed]
            min_skill = window[0][1]
            max_skill = window[-1][1]
            
            if max_skill - min_skill <= skill_tolerance:
                # Found a balanced match!
                matched_players = [p[0] for p in window]
                
                # Remove from queue
                pipe = self.r.pipeline()
                for pid in matched_players:
                    pipe.zrem(queue_key, pid)
                pipe.execute()
                
                return matched_players
        
        return None  # No balanced match yet, keep waiting
    
    def create_match(self, players: list[str], region: str) -> str:
        """Assign players to a game server and return match_id"""
        match_id = str(uuid.uuid4())
        
        # Pick least-loaded game server in region
        server = self.pick_game_server(region)
        
        # Notify game server to start match
        server.start_match(match_id, players)
        
        # Notify each player of match details (game server IP:port)
        for player_id in players:
            notify_player(player_id, {
                "match_id": match_id,
                "server_ip": server.ip,
                "server_port": server.game_port,
                "connect_token": generate_connect_token(player_id, match_id)
            })
        
        return match_id
```

### 2. Game Loop (Server-Side Tick)

```python
import asyncio
import time

class GameServer:
    def __init__(self, match_id: str, players: list[str]):
        self.match_id = match_id
        self.players = {pid: PlayerState() for pid in players}
        self.tick_rate = 60      # 60 ticks per second (16ms per tick)
        self.tick_count = 0
        self.game_time = 0.0
    
    async def game_loop(self):
        """Main game loop: process inputs → update state → send updates"""
        tick_interval = 1.0 / self.tick_rate  # 16.67ms
        next_tick = time.monotonic()
        
        while not self.is_game_over():
            # 1. Process all queued player inputs (received since last tick)
            await self.process_inputs()
            
            # 2. Update game state (physics, collisions, game rules)
            self.update_state(delta_time=tick_interval)
            
            # 3. Send state snapshot to all players
            await self.broadcast_state()
            
            self.tick_count += 1
            self.game_time += tick_interval
            
            # Sleep until next tick
            next_tick += tick_interval
            sleep_time = next_tick - time.monotonic()
            if sleep_time > 0:
                await asyncio.sleep(sleep_time)
            else:
                # We're behind schedule — log warning (tick took too long)
                metrics.increment("game.tick.overrun")
    
    def update_state(self, delta_time: float):
        """Authoritative physics simulation"""
        for player_id, state in self.players.items():
            # Apply velocity to position (server-authoritative)
            state.position.x += state.velocity.x * delta_time
            state.position.y += state.velocity.y * delta_time
            state.position.z += state.velocity.z * delta_time
            
            # Check collisions, boundaries
            self.apply_world_bounds(state)
            self.check_player_collisions(player_id, state)
    
    async def broadcast_state(self):
        """
        Send compressed game state to all players.
        Use delta encoding: only send what changed since player's last ack.
        """
        for player_id in self.players:
            snapshot = self.create_snapshot_for_player(player_id)
            await self.send_udp(player_id, snapshot)
    
    def create_snapshot_for_player(self, player_id: str) -> bytes:
        """
        Snapshot encoding:
        - Full state every 10 ticks (keyframe, for recovery)
        - Delta state other ticks (only changed values)
        - Interest management: only send nearby players' state
        """
        # Interest management: only show players within this player's view radius
        player_pos = self.players[player_id].position
        visible_players = [
            pid for pid, state in self.players.items()
            if distance(player_pos, state.position) < VIEW_RADIUS
        ]
        
        snapshot = {
            "tick": self.tick_count,
            "your_state": self.players[player_id].to_dict(),
            "other_players": {
                pid: self.players[pid].to_dict()
                for pid in visible_players
                if pid != player_id
            }
        }
        
        return compress(encode_snapshot(snapshot))  # ~100-500 bytes per snapshot
```

### 3. Client-Side Prediction and Lag Compensation

```
Without prediction:
  Player presses W (move forward)
  → Packet travels to server (50ms latency)
  → Server processes move
  → Server sends updated position (50ms back)
  → Player sees movement 100ms after pressing W
  → Game feels sluggish/unresponsive

With client-side prediction:
  Player presses W
  → Client IMMEDIATELY updates local position (no wait)
  → Packet also sent to server
  → Server confirms or corrects position
  → If server disagrees (prediction error): smooth correction applied

Client-side prediction (pseudocode):
  1. Local state: apply player's input immediately (predict outcome)
  2. Keep a history of inputs + predicted states for last 500ms
  3. On receiving server snapshot:
     a. If server agrees → delete old history, start from new authoritative state
     b. If server disagrees (position mismatch > 5 cm):
        - Snap to server position
        - Re-apply all unconfirmed inputs since snapshot tick to get new predicted position
```

### 4. Lag Compensation for Hit Detection

```
Scenario:
  Player A (100ms RTT to server) fires at Player B at t=1000ms (client time)
  Packet arrives at server at t=1050ms (server time)
  
  But at server's t=1050, Player B has already moved to a different position!
  Was the shot fair? Yes — from Player A's perspective, they aimed correctly.

Server-side lag compensation:
  1. Server keeps a history of all player positions for last 500ms
  2. When a "fire" event arrives at server time T with client timestamp Tc:
     a. Rewind server state to T - (T - Tc) = Tc (the time the player fired)
     b. Check hit detection against the REWOUND positions
     c. If hit: apply damage even though Player B has since moved

History storage per match:
  - 50 players × 60 ticks/sec × 500ms = 50 × 30 = 1500 position snapshots in RAM
  - Each snapshot: position(3 floats) + rotation(3 floats) = 24 bytes per player
  - Total: 1500 × 24 = 36 KB per match — trivially small
```

```python
from collections import deque
import dataclasses

@dataclasses.dataclass
class PositionSnapshot:
    tick: int
    server_time: float
    position: tuple[float, float, float]
    rotation: tuple[float, float, float]

class HitDetector:
    def __init__(self, history_length: int = 30):  # 500ms at 60Hz
        self.player_history: dict[str, deque[PositionSnapshot]] = {}
        self.history_length = history_length
    
    def record_position(self, player_id: str, snapshot: PositionSnapshot):
        if player_id not in self.player_history:
            self.player_history[player_id] = deque(maxlen=self.history_length)
        self.player_history[player_id].append(snapshot)
    
    def check_hit(self, shooter_id: str, target_id: str, 
                  shoot_tick: int, ray_origin: tuple, ray_direction: tuple) -> bool:
        """
        Check if a shot at `shoot_tick` hit the target.
        Uses rewound position of target at that tick.
        """
        history = self.player_history.get(target_id, [])
        
        # Find target's position at shoot_tick
        target_snapshot = None
        for snap in history:
            if snap.tick == shoot_tick:
                target_snapshot = snap
                break
        
        if not target_snapshot:
            return False  # Too old, can't verify
        
        # Check if ray from shooter intersects target hitbox at rewound position
        return ray_intersects_hitbox(ray_origin, ray_direction, 
                                      target_snapshot.position,
                                      hitbox_radius=0.5)  # 0.5m radius
```

---

## State Synchronization at Scale

```
One game server per match (not microservices per player):
  Reason: the game state is tightly coupled. Splitting it across services
  creates network overhead for every collision check, hit detection, item pickup.
  One server owns the full authoritative game state.

Server sizing:
  - 100-player battle royale: ~4 CPU cores, 2 GB RAM per match server
  - Game server is CPU-bound (physics simulation)
  - Each game server runs 1-4 simultaneous matches (depending on CPU)

Match server fleet:
  - 10M concurrent matches × 1 server per match = 10M servers?! No.
  - 10M matches × 50 players × 1 game server per 4 matches = 2.5M game servers
  - In practice: VMs with multiple game server processes, burst capacity on spot instances
  - Auto-scale based on matchmaking queue depth

Between-match data (persistent):
  Player stats, inventory, currency → PostgreSQL / DynamoDB (not on game server)
  Session state during match → game server RAM only (with checkpoint to S3)
```

---

## Key Design Decisions

| Decision | Choice | Trade-off |
|---|---|---|
| Transport | UDP for game state, TCP for events | UDP: low latency + packet loss OK for positions; TCP: reliable for critical events |
| Client prediction | Yes (for movement) | Better UX; requires reconciliation with server corrections |
| Tick rate | 60 Hz (server), 20 Hz (send to clients) | Simulation at 60Hz accuracy; bandwidth-efficient transmission at 20Hz |
| Interest management | Only send visible players' state | Reduces bandwidth from O(N²) to O(N × visible) |
| Hit detection | Server-authoritative + lag compensation | Fair for high-latency players; prevents miss-on-server-but-hit-on-client exploits |

---

## Common Interview Mistakes

1. **Using HTTP REST for game state updates**: REST adds 100-200ms per request. Game state updates (20/sec per player) need persistent UDP/WebSocket connections.

2. **Not addressing client-side prediction**: without it, all movement feels like it has 100ms latency. This is the most important UX technique in real-time games.

3. **Client-authoritative positions**: "the client sends its position to the server" — this allows speed hacks and wall hacks. Server must be authoritative.

4. **Ignoring lag compensation**: players on high-latency connections would never be able to hit anyone. Server-side lag compensation is required for fair hit detection.

5. **One game state database for all matches**: game state during a match belongs on the game server's RAM, not in a database. Database writes for every position update at 20 Hz × 50 players × 10M matches would be astronomical.

6. **Not discussing interest management**: "send all 100 players' positions to all 100 players" = 100 × 100 = 10,000 updates/tick per match. Interest management reduces this to only nearby players (~10-20).

7. **Ignoring matchmaking complexity**: just throwing players into a match without balancing skill ratings leads to poor game experience. Mention ELO/Glicko rating and skill-based matchmaking.

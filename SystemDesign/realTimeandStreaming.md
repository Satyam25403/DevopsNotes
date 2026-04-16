# Real-Time Systems in System Design

> A deep reference covering WebRTC, live video streaming (HLS/DASH), real-time leaderboards, and presence systems — the building blocks of latency-sensitive, large-scale applications.

---

## Table of Contents

1. [Fundamentals of Real-Time](#fundamentals)
2. [Transport Protocols](#transport-protocols)
3. [WebRTC](#webrtc)
4. [Live Video Streaming — HLS & DASH](#live-video-streaming)
5. [Real-Time Leaderboards](#real-time-leaderboards)
6. [Presence Systems](#presence-systems)
7. [Cross-Cutting Concerns](#cross-cutting-concerns)
8. [Interview Cheat Sheet](#interview-cheat-sheet)

---

## 1. Fundamentals of Real-Time {#fundamentals}

### What "Real-Time" Actually Means

| Class | Latency Target | Examples |
|---|---|---|
| Hard real-time | < 1 ms | Embedded control systems |
| Soft real-time | 10 – 100 ms | Live gaming, video calls |
| Near real-time | 100 ms – 1 s | Chat, presence, leaderboards |
| Streaming real-time | 1 – 30 s | HLS/DASH live video |

In distributed systems, "real-time" almost always means *near real-time* or *soft real-time*.

### Core Tradeoffs

```
Consistency  ←————————————→  Availability
      Strong CP systems        Eventual AP systems
  (SQL transactions)         (Cassandra, Redis)

Low Latency  ←————————————→  Durability
  In-memory fanout             Write-ahead logs, Kafka
```

**CAP Theorem Angle for Real-Time:** Most real-time systems prioritize *AP* (Availability + Partition Tolerance). A leaderboard showing a slightly stale rank is better than a leaderboard that's down.

---

## 2. Transport Protocols {#transport-protocols}

### HTTP/1.1 Polling vs Long-Polling

```
Short Polling:   Client ──GET──► Server (200 + data or 204 empty)
                 Client ──GET──► Server   (repeated every N seconds)

Long Polling:    Client ──GET──► Server ··· holds connection ···
                                           ──200 when event ready──►
                 Client ──GET──► Server   (immediately reconnects)
```

**Use when:** Legacy clients, firewalls blocking WebSockets, low event frequency.  
**Avoid when:** > 10k concurrent clients (thundering herd on reconnect).

---

### Server-Sent Events (SSE)

- Unidirectional: server → client only
- HTTP/1.1 `Content-Type: text/event-stream`
- Auto-reconnect built into browser EventSource API
- Works through most proxies/CDNs (unlike raw WebSocket)

```
GET /events HTTP/1.1
Accept: text/event-stream

HTTP/1.1 200 OK
Content-Type: text/event-stream

data: {"type":"score","userId":42,"score":9800}\n\n
data: {"type":"presence","userId":7,"status":"online"}\n\n
```

**Best for:** Live feeds, leaderboard updates, dashboards, notifications. Not suitable for bidirectional communication.

---

### WebSocket

- Full-duplex, persistent TCP connection
- Upgrade handshake via HTTP 101 Switching Protocols
- Low framing overhead after upgrade (~2–10 bytes/frame)

```
Client                         Server
  ──── HTTP GET (Upgrade) ────►
  ◄─── 101 Switching Protocol ──
  ──── WS Frame (message) ─────►
  ◄─── WS Frame (push) ─────────
         [persistent connection]
```

**Best for:** Chat, collaborative editing, gaming, bidirectional signaling.  
**Scaling challenge:** Stateful connections require sticky sessions or a shared pub-sub layer (Redis).

---

### gRPC Streaming

- Bidirectional or server-streaming over HTTP/2
- Strong typing via Protocol Buffers
- Native load balancing challenges (L7 needed)

```proto
service Leaderboard {
  rpc WatchRank (WatchRequest) returns (stream RankUpdate);
  rpc StreamGame (stream GameEvent) returns (stream GameState);
}
```

**Best for:** Internal microservice streaming, mobile apps, when strong schemas matter.

---

### QUIC / WebTransport

- UDP-based, multiplexed streams without head-of-line blocking
- Sub-connection streams: packet loss in one stream doesn't stall others
- WebTransport API gives browsers low-latency datagram + stream access
- Emerging standard — still gaining browser/proxy support (2024+)

---

## 3. WebRTC {#webrtc}

### Architecture Overview

WebRTC enables peer-to-peer (P2P) audio, video, and data between browsers without a media server — but always requires a signaling server and usually STUN/TURN.

```
           ┌─────────────────────────────┐
           │       Signaling Server       │
           │  (WebSocket / SIP / custom)  │
           └──────────┬──────────────────┘
                      │ SDP offer/answer
              ┌───────┴──────────┐
              │                  │
           Peer A              Peer B
              │                  │
              └────── Media ─────┘
              (SRTP/DTLS over UDP or TCP)
```

### Key Components

**ICE (Interactive Connectivity Establishment)**
- Collects *candidates*: host, server-reflexive (via STUN), relayed (via TURN)
- Tries all candidate pairs, selects best one
- Handles NAT traversal

**STUN (Session Traversal Utilities for NAT)**
- Lightweight: lets peers discover their public IP/port
- Free servers available (Google: `stun.l.google.com:19302`)

**TURN (Traversal Using Relays around NAT)**
- Fallback relay server when P2P fails (~15–20% of connections)
- Expensive: all media traffic flows through it
- Deploy regionally to minimize latency

**SDP (Session Description Protocol)**
- Text blob describing codecs, bandwidth, ICE candidates
- Exchanged out-of-band via your signaling server

```
Offer/Answer Flow:

Peer A                  Signaling               Peer B
  ── createOffer() ──►
  ── setLocalDesc() ──►
  ── send(offer) ─────────────────────────────►
                                      ◄── createAnswer()
                                      ◄── setLocalDesc()
  ◄────────────────────── send(answer) ────────
  ── setRemoteDesc() ──►
                                      ICE candidate exchange
  ◄══════════════ P2P Media Connected ═════════►
```

### Codecs

| Media | Codec | Notes |
|---|---|---|
| Audio | Opus | Adaptive bitrate 6–510 kbps, mandatory in WebRTC |
| Video | VP8/VP9 | Google-backed, royalty-free |
| Video | H.264 | Hardware-accelerated on most devices |
| Video | AV1 | Best compression, higher CPU cost |

### Scaling WebRTC Beyond P2P

**Mesh (P2P):** Works for ≤ 4 participants. Each peer sends N-1 streams. O(N²) bandwidth.

```
A ←──────────► B
 ↖           ↗
   ↘       ↙
      C ←─► D
```

**SFU (Selective Forwarding Unit):** Media server receives all streams, forwards selectively. Clients send 1 stream, receive N-1 streams. Used by: Zoom, Discord, Teams, Daily.co.

```
A ──► SFU ──► B
B ──► SFU ──► A
C ──► SFU ──► A, B
```

**MCU (Multipoint Control Unit):** Server decodes, mixes, re-encodes into one stream per participant. Low client bandwidth, high server CPU. Rarely used today.

### WebRTC Data Channels

- SCTP over DTLS over UDP
- Ordered or unordered delivery
- Reliable or unreliable (like UDP)
- Use for: game state sync, file sharing, collaborative cursors

```javascript
const dc = peerConnection.createDataChannel("game", {
  ordered: false,      // UDP-like, drop old packets
  maxRetransmits: 0    // Never retransmit
});
```

### WebRTC Failure Points and Mitigations

| Failure | Mitigation |
|---|---|
| ICE failure (symmetric NAT) | Always deploy TURN fallback |
| Packet loss causing video artifacts | Adaptive bitrate + FEC + NACK |
| High CPU from encoding | Hardware-accelerated codecs (H.264 HW) |
| Signaling server becomes SPOF | Deploy signaling in multiple regions |
| TURN overload | Scale TURN horizontally, use Coturn + load balancer |

---

## 4. Live Video Streaming — HLS & DASH {#live-video-streaming}

### The Streaming Pipeline

```
Camera/Encoder
     │
     ▼
RTMP Ingest ──► Transcoding ──► Packaging ──► CDN ──► Player
(OBS, FFmpeg)   (multiple        (HLS/DASH      (Edge    (hls.js,
                 bitrates)        segments)    Nodes)    dash.js)
```

### HLS (HTTP Live Streaming)

Created by Apple. Segments video into `.ts` (MPEG-TS) or `.fmp4` chunks, served via standard HTTP.

```
master.m3u8         (variant playlist — bitrate ladder)
   ├── 1080p.m3u8   (index — lists segments)
   │     ├── seg0001.ts
   │     ├── seg0002.ts
   │     └── ...
   ├── 720p.m3u8
   └── 360p.m3u8
```

**Master Playlist:**
```
#EXTM3U
#EXT-X-STREAM-INF:BANDWIDTH=5000000,RESOLUTION=1920x1080
1080p.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=2500000,RESOLUTION=1280x720
720p.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=800000,RESOLUTION=640x360
360p.m3u8
```

**Segment Playlist:**
```
#EXTM3U
#EXT-X-VERSION:3
#EXT-X-TARGETDURATION:6
#EXTINF:6.006,
seg0001.ts
#EXTINF:6.006,
seg0002.ts
```

**Latency:** Standard HLS = 15–30 s. Low-Latency HLS (LL-HLS, RFC 8216bis) = 1–3 s using partial segments and HTTP/2 push.

---

### DASH (Dynamic Adaptive Streaming over HTTP)

Open standard by MPEG. More flexible than HLS — codec-agnostic, supports DRM natively.

```xml
<!-- MPD (Media Presentation Description) -->
<MPD type="dynamic">
  <AdaptationSet mimeType="video/mp4">
    <Representation id="1080p" bandwidth="5000000" width="1920" height="1080">
      <SegmentTemplate media="seg$Number$.mp4" duration="6"/>
    </Representation>
    <Representation id="720p" bandwidth="2500000" ...>
    </Representation>
  </AdaptationSet>
</MPD>
```

**HLS vs DASH:**

| Feature | HLS | DASH |
|---|---|---|
| Creator | Apple | MPEG consortium |
| Format | MPEG-TS / fMP4 | fMP4 only |
| Codec | H.264, H.265, AAC | Any (H.264, VP9, AV1...) |
| DRM | FairPlay (native) | Widevine, PlayReady, ClearKey |
| Native iOS/Safari | ✅ | ❌ (needs dash.js) |
| Low latency | LL-HLS (~2s) | DASH-LL (~2s) |
| Complexity | Simpler | More flexible |

---

### Adaptive Bitrate (ABR) Logic

Player monitors:
- Buffer level (seconds of video buffered)
- Estimated download bandwidth (EWMA of segment fetch times)
- CPU/decoding capability

```
Buffer  < 5s   → Drop quality immediately
Buffer 5–15s   → Maintain current quality
Buffer > 15s + BW > next tier → Upgrade quality
```

Common ABR algorithms: BOLA (buffer-occupancy-based), FESTIVE, MPC (model predictive control).

---

### CDN Architecture for Live Video

```
Origin (Packager)
    │
    ├──► CDN PoP (Region 1) ──► Viewers
    ├──► CDN PoP (Region 2) ──► Viewers
    └──► CDN PoP (Region 3) ──► Viewers
```

**Cache-Control strategy:**
- Segments (`.ts`, `.mp4`): `Cache-Control: public, max-age=86400` (immutable once written)
- Playlists (`.m3u8`, `.mpd`): `Cache-Control: public, max-age=2` (must refresh for live)
- Never cache playlists too long — that's the primary source of player lag behind live edge.

**Live vs VOD:**

| Concern | Live | VOD |
|---|---|---|
| Playlist caching | Very short TTL (2–5s) | Long TTL |
| Segment availability | Rolling window | Permanent |
| CDN origin load | Constant | Burst on release |
| Error recovery | Critical (stream dies) | Easy (retry) |

---

### Latency Reduction Techniques

| Technique | Reduction | Tradeoff |
|---|---|---|
| Shorter segments (2s vs 6s) | ~10s | More HTTP requests, higher origin load |
| LL-HLS / DASH-LL partial segments | Down to 1–3s | Complex implementation |
| HTTP/2 server push of next segment | Saves 1 RTT | Server complexity |
| Chunked transfer encoding | Start playing before segment complete | Buffer edge cases |
| Reduce DVR window | Lower memory at edge | No rewind |

---

## 5. Real-Time Leaderboards {#real-time-leaderboards}

### Core Data Structure: Redis Sorted Sets

Redis Sorted Set (`ZSET`) is the canonical building block.

```
ZADD leaderboard:game1 9800 "user:42"   # O(log N)
ZADD leaderboard:game1 9500 "user:17"
ZINCRBY leaderboard:game1 200 "user:42" # Increment score

ZRANK leaderboard:game1 "user:42"       # Rank (0-indexed, asc)
ZREVRANK leaderboard:game1 "user:42"    # Rank (0-indexed, desc)
ZREVRANGE leaderboard:game1 0 9 WITHSCORES  # Top 10
ZRANGEBYSCORE leaderboard:game1 9000 +inf   # Score range
```

Complexity: O(log N) insert/update, O(log N + K) for range queries.

---

### Tiered Leaderboard Architecture

For millions of players, a single sorted set hits limits. Use tiered sharding:

```
Global Leaderboard (top 10k)
       ├── Shard 0: users A–F
       ├── Shard 1: users G–M
       ├── Shard 2: users N–S
       └── Shard 3: users T–Z

Friends Leaderboard (per user's social graph)
       └── Stored per user or computed on read
```

**Reading rank in sharded setup:**
1. Get user's score: O(1) from user profile
2. Count users with score > X: query each shard in parallel → sum results
3. User's rank = total count + 1

---

### Real-Time Update Fan-out

```
Score Event
    │
    ▼
Score Service ──► Redis ZINCRBY ──► Pub/Sub Channel "lb:updated"
                                           │
                              ┌────────────┘
                              ▼
                    WebSocket Server (horizontally scaled)
                    Subscribed to Redis Pub/Sub
                              │
                    ┌─────────┼─────────┐
                    ▼         ▼         ▼
                 Client    Client    Client
```

**Broadcast vs targeted updates:**
- Top 100 updates → broadcast to all viewers of leaderboard
- Personal rank updates → push only to affected user's socket
- Avoid recalculating full leaderboard on every score change

---

### Eventual Consistency Strategies

**Batch aggregation (write-heavy games):**
- Accept score events into Kafka
- Aggregator consumes, batches per user per 1s window
- Apply ZINCRBY in batch — fewer round trips
- Trades ~1s staleness for 100x write throughput

**Read-your-writes consistency:**
- User's own score: serve from user session cache (immediate)
- Others' ranks: serve from Redis (near real-time)
- Leaderboard top-N: cache at 1s or 5s intervals

---

### Database Schema (Persistence Layer)

```sql
CREATE TABLE scores (
  user_id    BIGINT NOT NULL,
  game_id    BIGINT NOT NULL,
  score      BIGINT NOT NULL,
  updated_at TIMESTAMP NOT NULL,
  PRIMARY KEY (game_id, score DESC, user_id)  -- Composite for range queries
);

CREATE INDEX idx_scores_user ON scores(user_id, game_id);
```

Redis is the hot path. PostgreSQL (or Cassandra for extreme write scale) serves as durable backup for recovery and audit.

---

### Leaderboard Variants

| Type | Implementation |
|---|---|
| Global all-time | Single ZSET per game |
| Daily/weekly | ZSET with key `lb:game1:2024-01-15`, TTL auto-expiry |
| Friends | ZINTERSTORE of friend user ZSETs (expensive at scale) |
| Regional | Separate ZSET per region, merged for global |
| Percentile rank | `ZRANK / ZCARD` → percentile |

---

## 6. Presence Systems {#presence-systems}

### What Presence Tracks

- **Online/offline status** (coarse: is the user reachable?)
- **Active/idle/away** (fine: is the user engaging?)
- **Location/context** (typing in channel X, viewing document Y)
- **Device** (mobile vs desktop, multiple sessions)

---

### Architecture Patterns

#### Pattern 1: Heartbeat + TTL (Redis)

```
Client ──── heartbeat every 30s ──► Presence Service
                                        │
                                   SET user:42:presence "online" EX 60
                                        │
                                   (expires in 60s if no heartbeat)
```

- Online = key exists in Redis
- Offline = key missing (expired or explicit DEL)
- TTL is your "graceful disconnect window"
- Trade-off: 30–60s lag on disconnect detection

---

#### Pattern 2: Connection Lifecycle (WebSocket events)

```
Connect  event  ──► SET user:42:status "online"
                    PUBLISH presence:channel42 '{"user":42,"status":"online"}'

Disconnect event ──► DEL user:42:status
                     PUBLISH presence:channel42 '{"user":42,"status":"offline"}'
```

- Instant offline detection
- But: server crashes → no disconnect event → ghost presences
- Mitigation: combine with TTL heartbeat as safety net

---

#### Pattern 3: Fanout via Pub/Sub

```
User 42 goes online
    │
    ▼
Presence Service
    ├── Updates Redis
    └── Publishes to channels:
          - user:42:friends  (notify all friends)
          - room:sports      (if user is in this room)
          - doc:abc123       (if user has document open)
                    │
              Channel subscribers receive push
              and update their local presence state
```

**Fanout cost:** A user with 5,000 followers going online = 5,000 presence messages. Mitigations:
- Lazy presence: only push to active viewers, not all followers
- Presence polling fallback for large follow counts
- Aggregate: batch all presence events in 1s windows

---

### Presence at Scale

**Multi-device presence:**
```
user:42:sessions (Redis SET)
  → "session:abc" (device 1)
  → "session:xyz" (device 2)

User is "online" if SET has any members.
User is "offline" only when SET is empty.
```

**Geo-distributed presence:**
```
Region A Cluster ──► Global Presence Bus (Kafka)
Region B Cluster ──► Global Presence Bus
Region C Cluster ──► Global Presence Bus
                            │
                     Global aggregator
                     writes to global Redis
```

- Local presence queries hit regional Redis (low latency)
- Cross-region presence subscribes via Kafka (higher latency, ~100ms)
- Use CRDTs (Conflict-free Replicated Data Types) for multi-region merging

---

### Typing Indicators

Special case of fine-grained presence:

```
User starts typing:
  Client ──► POST /typing/start  (or WS frame)
  Server ──► PUBLISH room:42:typing '{"user":7,"typing":true}'
  Server ──► SET typing:room42:user7 "1" EX 5   (5s TTL)

User stops typing:
  After 5s of inactivity — key expires → server detects → publish stop
  OR client sends explicit stop event
```

Throttle: client should send at most 1 typing event per 2 seconds (debounce).

---

### Read Receipts

```
Message delivered event:
  Receiver ──► POST /messages/456/receipt  {status: "delivered"}
  Server   ──► UPDATE messages SET delivered_at = NOW() WHERE id = 456
  Server   ──► PUBLISH chat:thread1 '{"msgId":456,"status":"delivered","userId":9}'
  Sender   ◄── WS push: message 456 delivered

Message seen event:
  When receiver scrolls to message in viewport:
  Receiver ──► POST /messages/456/receipt  {status: "seen"}
```

---

## 7. Cross-Cutting Concerns {#cross-cutting-concerns}

### Connection Management at Scale

**The C10k → C1M problem:**  
- HTTP long-polling: 1 thread/connection → can't scale past ~10k
- Async I/O (Node.js, Go, Netty, Nginx): 1 thread, many connections via event loop
- Target: 100k–1M concurrent WS connections per server node

**Load balancing WebSockets:**
```
Client ──► L4 Load Balancer (TCP passthrough)
                 │
          Sticky session (consistent hash on user_id)
                 │
          WS Server Node (stateful)
                 │
          Redis Pub/Sub (shared state)
```

Use consistent hashing so a user's socket always routes to the same server. Fallback: store session mapping in Redis.

---

### Backpressure

Real-time systems must handle slow consumers:

```
Producer (fast) ──► Buffer ──► Consumer (slow)
                      │
              if buffer fills:
              - Drop oldest (leaky bucket)
              - Drop newest (backpressure signal)
              - Slow producer (TCP flow control)
```

WebSocket servers should have per-connection send buffers with eviction policies. Kafka consumers use offset management to handle lag.

---

### Monitoring Real-Time Systems

| Metric | Alert Threshold |
|---|---|
| WebSocket connection count | > 80% of server capacity |
| Message delivery latency (p99) | > 200ms for presence/chat |
| Leaderboard update lag | > 1s for top-10 |
| Video player buffer health | < 5s buffer |
| Video startup time | > 3s |
| TURN relay ratio | > 25% (implies NAT issues) |
| ICE failure rate | > 5% |
| Presence TTL miss rate | > 0.1% |

---

### Security Considerations

**WebSocket:**
- Always use `wss://` (TLS)
- Validate `Origin` header on upgrade to prevent CSWSH (cross-site WebSocket hijacking)
- Authenticate via token in first WS message (not in URL — URLs log)
- Rate-limit message sends per connection

**WebRTC:**
- All media is SRTP (mandatory in WebRTC spec)
- Validate signaling origin; don't trust SDP blindly
- TURN server: authenticate with credential rotation

**Leaderboards:**
- Sign score submissions with server-side validation (anti-cheat)
- Rate-limit score events per user per second
- Audit log high-score jumps

**Presence:**
- Never expose presence of blocked users
- Respect "invisible" mode: update internal state, don't publish externally
- Throttle presence subscription requests (DoS vector)

---

## 8. Interview Cheat Sheet {#interview-cheat-sheet}

### Quick Decision Framework

```
What transport should I use?
├── Bidirectional, low-latency? ──────────────────► WebSocket
├── Server-to-client only, simple? ───────────────► SSE
├── P2P media (video/audio/data)? ────────────────► WebRTC
├── Live video broadcast to many? ────────────────► HLS / DASH
└── Internal service streaming? ──────────────────► gRPC streaming

What data store for real-time rank?
├── < 1M users, single game? ─────────────────────► Single Redis ZSET
├── > 1M users? ──────────────────────────────────► Sharded ZSETs
└── Time-windowed (daily/weekly)? ────────────────► ZSETs with TTL

What presence pattern?
├── Instant detection needed? ────────────────────► WebSocket lifecycle + heartbeat
└── Coarse detection OK (30s)? ───────────────────► Heartbeat TTL only
```

### Capacity Estimation Examples

**Leaderboard (10M users, game updates 100/s globally):**
- Redis ZINCRBY: ~0.1ms per op → 100 ops/s → trivial for single Redis
- Fan-out to 50k leaderboard viewers: Pub/Sub → WS servers → clients
- Memory: 10M entries × ~64 bytes ≈ 640 MB → fits in one Redis node

**Presence (50M users, 5% active = 2.5M online):**
- Heartbeat load: 2.5M × (1/30s) ≈ 83k Redis ops/s → needs Redis cluster
- WS connections: 2.5M sockets / 50k per node = 50 WS server nodes
- Pub/Sub fan-out: if avg user has 200 friends → 2.5M events/s on connect wave (use lazy delivery)

**Live Video (100k concurrent viewers):**
- Segment requests: 100k × (1 req/6s) ≈ 17k req/s → CDN handles easily
- Origin load: only cache misses → ~1-5% → ~170–850 req/s to origin
- Bandwidth: 100k × 2 Mbps avg = 200 Gbps → multi-CDN required




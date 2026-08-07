# Real-Time Updates — System Design Pattern

## Table of Contents
1. [The Problem](#the-problem)
2. [The Two-Hop Model](#the-two-hop-model)
3. [Hop 1: Client-Server Protocols](#hop-1-client-server-protocols)
   - [Simple Polling](#1-simple-polling)
   - [Long Polling](#2-long-polling)
   - [Server-Sent Events (SSE)](#3-server-sent-events-sse)
   - [WebSockets](#4-websockets)
   - [WebRTC](#5-webrtc)
   - [Protocol Comparison](#protocol-comparison)
4. [Hop 2: Source-to-Server Delivery](#hop-2-source-to-server-delivery)
   - [Simple Polling from Source](#1-simple-polling-from-source)
   - [Consistent Hashing](#2-consistent-hashing)
   - [Pub/Sub](#3-pubsub-most-common)
5. [Scaling Real-Time Systems](#scaling-real-time-systems)
   - [Connection Storage](#connection-storage)
   - [Fan-out Strategies](#fan-out-strategies)
   - [Thundering Herd](#thundering-herd)
6. [Failure Handling](#failure-handling)
   - [Client Reconnection](#client-reconnection)
   - [Message Ordering](#message-ordering)
   - [Server Failures](#server-failures)
7. [Networking Fundamentals](#networking-fundamentals)
8. [Real-World Examples](#real-world-examples)
9. [Interview Cheat Sheet](#interview-cheat-sheet)

---

## The Problem

Standard HTTP follows a **request-response** model:
- Client sends request
- Server responds
- Connection closes

This breaks down when the server needs to **proactively push updates** to clients. Examples:

| Product | Real-Time Need |
|---------|---------------|
| Cricbuzz | Live score every ball |
| Google Docs | Keystrokes from other users in milliseconds |
| WhatsApp | Message delivery without polling |
| Robinhood | Stock price changes |
| Uber | Driver location updates |
| Ticketmaster | Seat availability during high-demand sale |
| FB Live | Comments appearing in real time |

---

## The Two-Hop Model

This is the most important framework. Most candidates only think about Hop 1.

```
                    HOP 2                      HOP 1
┌────────┐    ┌──────────────┐    ┌──────────────┐    ┌────────┐
│ Score  │───▶│   Kafka /    │───▶│  SSE/WS      │───▶│ Client │
│ Source │    │   Redis      │    │  Server      │    │Browser │
└────────┘    └──────────────┘    └──────────────┘    └────────┘

Hop 1: How does the update get from server to client?
Hop 2: How does the update get from the source to the right server?
```

Both hops need separate design decisions.

---

## Hop 1: Client-Server Protocols

### 1. Simple Polling

Client hits the server repeatedly on a fixed interval.

```
Client ──GET /updates──▶ Server ──"No data"──▶ Client  (connection closes)
Client ──GET /updates──▶ Server ──"No data"──▶ Client  (connection closes)
Client ──GET /updates──▶ Server ──"Data!"───▶ Client   (connection closes)
```

**How it works:**
```js
setInterval(async () => {
  const res = await fetch('/api/score');
  const data = await res.json();
  render(data);
}, 3000); // poll every 3 seconds
```

**Pros:**
- Simplest to implement
- Works everywhere, no special server support
- Stateless server

**Cons:**
- Latency = polling interval (3s poll = up to 3s delay)
- Wasteful — most responses are empty
- Hammers the server even when nothing changes

**When to use:** Very infrequent, non-critical updates (e.g., refresh a dashboard every 30s)

---

### 2. Long Polling

Client sends a request and server **holds it open** until data is available, then closes and client immediately reconnects.

```
Client ──GET /updates──▶ Server (holds connection open...)
                                │
                         event happens
                                │
                         Server ──"Data!"──▶ Client  (connection closes)
Client immediately reconnects ──GET /updates──▶ Server (holds again...)
```

**How it works:**
```js
// Client
async function longPoll() {
  const res = await fetch('/api/updates'); // hangs until server responds
  const data = await res.json();
  render(data);
  longPoll(); // immediately reconnect
}

// Server
app.get('/api/updates', async (req, res) => {
  const data = await waitForUpdate(); // blocks until event occurs
  res.json(data);
});
```

**Pros:**
- Near real-time (response sent immediately when event occurs)
- Works through firewalls and proxies (standard HTTP)
- No special client support needed

**Cons:**
- High reconnect overhead
- Hard to scale — each waiting request holds a thread/connection
- Complex timeout handling (what if no event for 30s?)
- Not efficient for high-frequency updates

**When to use:** When SSE/WebSocket is not possible, or updates are infrequent but must be immediate

---

### 3. Server-Sent Events (SSE)

Client opens a single HTTP connection that **stays open forever**. Server streams events as they happen.

```
Client ──GET /events──▶ Server
                        │
                        ├──event 1──▶ Client
                        ├──event 2──▶ Client
                        ├──event 3──▶ Client
                        │  (connection stays open)
```

**Wire format** — plain text, newline-delimited:
```
id: 101
event: score
data: {"runs": 245, "wickets": 3, "over": "32.4"}

id: 102
event: wicket
data: {"player": "Kohli", "runs": 67, "bowler": "Cummins"}

retry: 3000

```
Each event ends with a blank line (`\n\n`).

**Server implementation:**
```js
app.get('/events', (req, res) => {
  res.setHeader('Content-Type', 'text/event-stream');
  res.setHeader('Cache-Control', 'no-cache');
  res.setHeader('Connection', 'keep-alive');

  const lastId = req.headers['last-event-id'];

  // replay missed events on reconnect
  if (lastId) {
    const missed = getMissedEventsSince(lastId);
    missed.forEach(e => {
      res.write(`id: ${e.id}\ndata: ${JSON.stringify(e.data)}\n\n`);
    });
  }

  // subscribe to live events
  const unsubscribe = subscribe('match:IND-AUS', (event) => {
    res.write(`id: ${event.id}\ndata: ${JSON.stringify(event)}\n\n`);
  });

  req.on('close', () => unsubscribe());
});
```

**Client implementation:**
```js
const es = new EventSource('/events');

es.onmessage = (e) => render(JSON.parse(e.data));

es.addEventListener('wicket', (e) => {
  showWicketAnimation(JSON.parse(e.data));
});

// browser auto-reconnects with Last-Event-ID header
// no manual reconnect logic needed
```

**Reconnection flow:**
```
Connection drops
      ↓
Browser waits retry ms (default ~3000ms, override with retry: field)
      ↓
GET /events HTTP/1.1
Last-Event-ID: 101        ← browser adds automatically
      ↓
Server replays events 102, 103... then continues live stream
```

**Pros:**
- Simple — plain HTTP, works through proxies
- Built-in reconnection with `Last-Event-ID`
- Browser `EventSource` API handles everything
- HTTP/2 multiplexes multiple SSE streams over one TCP connection

**Cons:**
- One-directional only (server → client)
- Max 6 concurrent connections per domain in HTTP/1.1 (not an issue with HTTP/2)
- Not ideal for bidirectional use cases

**When to use:** Live feeds, notifications, dashboards, scores — anything server-pushes, client-reads

---

### 4. WebSockets

Upgrades an HTTP connection into a **full-duplex TCP tunnel**. Both sides can send at any time.

```
Client ──HTTP Upgrade──▶ Server
       ◀──101 Switching Protocols──
       ◀══════ persistent TCP tunnel ══════▶
       ◀── server pushes ──
       ── client sends ──▶
```

**Handshake:**
```http
GET /ws HTTP/1.1
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==

HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

After this, it's raw TCP frames — not HTTP anymore.

**Server implementation:**
```js
const wss = new WebSocketServer({ port: 8080 });

wss.on('connection', (ws, req) => {
  const matchId = parseMatchId(req.url);

  // receive from client
  ws.on('message', (msg) => {
    const data = JSON.parse(msg);
    handleClientMessage(data, ws);
  });

  // subscribe and push to client
  const unsubscribe = subscribe(matchId, (event) => {
    ws.send(JSON.stringify(event));
  });

  ws.on('close', () => unsubscribe());
});
```

**Client implementation:**
```js
const ws = new WebSocket('wss://cricbuzz.com/ws/match/IND-AUS');

ws.onopen = () => ws.send(JSON.stringify({ type: 'subscribe', matchId: 'IND-AUS' }));
ws.onmessage = (e) => render(JSON.parse(e.data));
ws.onclose = () => reconnect(); // must handle reconnect manually
```

**Pros:**
- Full-duplex — client and server both send freely
- Low latency (no HTTP overhead per message after handshake)
- Great for chat, collaborative editing, multiplayer games

**Cons:**
- No built-in reconnection (must implement yourself)
- Harder to scale — load balancers need sticky sessions or a broker
- Stateful — proxies/firewalls can be tricky
- More complex than SSE for one-way use cases

**When to use:** Chat (WhatsApp), collaborative editing (Google Docs), multiplayer games, trading platforms

---

### 5. WebRTC

**Peer-to-peer** communication — data flows directly browser-to-browser, bypassing the server for media.

```
Browser A ──signaling──▶ Server ──signaling──▶ Browser B
Browser A ◀════════════ direct P2P media ═══════════▶ Browser B
```

**Pros:**
- Ultra-low latency for media (no server hop)
- Server doesn't bear media bandwidth cost

**Cons:**
- Complex setup (STUN/TURN servers, ICE candidates, SDP negotiation)
- Not suitable for server-to-client data push
- Overkill for most system design interview scenarios

**When to use:** Video/audio calls (Zoom, Google Meet), P2P file sharing

---

### Protocol Comparison

| | Simple Polling | Long Polling | SSE | WebSocket | WebRTC |
|--|--|--|--|--|--|
| Direction | Client→Server | Client→Server | Server→Client | Bidirectional | Peer-to-Peer |
| Latency | = poll interval | Near real-time | Near real-time | Near real-time | Lowest |
| Reconnect | Manual | Manual | Built-in | Manual | Manual |
| Complexity | Lowest | Low | Low | Medium | Highest |
| Scalability | Easy | Medium | Medium | Hard | Hard |
| HTTP compatible | Yes | Yes | Yes | Upgrade | Separate |
| Best for | Dashboards | Simple alerts | Feeds, scores | Chat, collab | Video/audio |

---

## Hop 2: Source-to-Server Delivery

Once the client-server protocol is chosen, you need to get the event **from the source to the correct server** that holds the client's connection.

```
┌─────────────┐         ┌─────────────────────────────┐
│Score Service│         │   10 SSE Servers             │
│             │──────?──▶  Server 1: [UserA, UserC]   │
│ "Wicket!"   │         │  Server 2: [UserB, UserD]   │
└─────────────┘         │  Server 3: [UserE]          │
                        └─────────────────────────────┘
```

How does the Score Service know which server to send to?

---

### 1. Simple Polling from Source

Each SSE server polls the DB/source periodically.

```
SSE Server 1 ──GET /latest-score──▶ DB (every 1s)
SSE Server 2 ──GET /latest-score──▶ DB (every 1s)
```

**Pros:** Simple, no extra infrastructure  
**Cons:** Latency = poll interval, DB gets hammered by N servers × poll rate

---

### 2. Consistent Hashing

Route users to servers deterministically based on a hash of their userId or matchId. Same user always lands on the same server.

```
hash(userId) % N → Server index

UserA → hash → Server 1  (always)
UserB → hash → Server 2  (always)

Score update for match → hash(matchId) → Server 1 → push to all users of that match
```

**Pros:** No broker needed, low latency  
**Cons:**
- Rebalancing is hard when servers are added/removed
- Hot spots if one match has 10x more users than others
- Server failure = reconnect storm to new server

---

### 3. Pub/Sub (Most Common)

A message broker (Redis, Kafka) sits between source and SSE servers. Each SSE server subscribes to channels it has active users for.

```
Score Service ──PUBLISH match:IND-AUS──▶ Redis/Kafka
                                              │
                          ┌───────────────────┤
                          ▼                   ▼
                    SSE Server 1        SSE Server 2
                    (subscribed)        (subscribed)
                          │                   │
                    push to              push to
                   [UserA, UserC]       [UserB, UserD]
```

**Redis Pub/Sub:**
```
PUBLISH match:IND-AUS '{"score":"245/3","over":"32.4","event":"boundary"}'
```

Each SSE server:
```js
redisClient.subscribe('match:IND-AUS', (message) => {
  const event = JSON.parse(message);
  localConnections.get('match:IND-AUS').forEach(res => {
    res.write(`data: ${message}\n\n`);
  });
});
```

**Redis vs Kafka:**

| | Redis Pub/Sub | Kafka |
|--|--|--|
| Persistence | No (ephemeral) | Yes (configurable) |
| Replay | No | Yes (by offset) |
| Throughput | Very high | Very high |
| Use case | Fire-and-forget | Replay, auditing, at-least-once |
| Missed events | Lost | Can replay on reconnect |

**When to use Kafka:** When clients need to replay missed events using `Last-Event-ID` (Kafka consumer can seek to offset).

---

## Scaling Real-Time Systems

### Connection Storage

SSE/WebSocket connections are **OS-level TCP sockets**. They cannot be serialized or stored in a database.

```
What CAN be stored in Redis:
  conn:{userId} → { serverId, matchId, connectedAt }   ← metadata only

What CANNOT be stored in Redis:
  The actual socket/response object                     ← lives in process RAM only
```

Each server keeps an in-memory map:
```js
// Server 1's local state (RAM only)
const connections = new Map();
// {
//   "match:IND-AUS": [res_userA, res_userC],
//   "match:ENG-NZ":  [res_userF]
// }
```

### Fan-out Strategies

**Small fan-out** (e.g., chat room, 100 users):
```
Pub/Sub → SSE Server → iterate local connections → push
```

**Large fan-out** (e.g., Kohli wicket, 50M users):
```
Score event → Kafka topic
           → 100s of consumer workers (each handles a shard of users)
           → each worker pushes to its SSE server shard
           → SSE server pushes to local connections
```

**Celebrity/hot key problem:**
- One match can have 10x more users than others
- Solution: over-partition (more Kafka partitions than servers), assign multiple partitions per hot match

### Thundering Herd

When a push notification fires and 10M users tap it simultaneously:

```
All users tap → all open app → all hit /scorecard API
             → cache expires at same moment
             → all requests miss cache → all hit DB → DB dies
```

**Solutions:**

1. **Jittered TTL** — randomize cache expiry
```js
const TTL = 60 + Math.floor(Math.random() * 10); // 60-70s, not all expire at once
```

2. **Cache locking (mutex)** — only one request rebuilds cache, others wait
```js
const lock = await redis.set('lock:scorecard', '1', 'NX', 'EX', 5);
if (lock) {
  const data = await db.query(...);
  await redis.setex('scorecard', 60, JSON.stringify(data));
} else {
  await waitForCache('scorecard');
}
```

3. **Request coalescing** — collapse identical in-flight requests into one DB call

---

## Failure Handling

### Client Reconnection

**SSE** — handled automatically by the browser:
```
1. Connection drops
2. Browser waits retry ms (default 3000ms)
3. Sends GET with Last-Event-ID header
4. Server replays missed events, resumes stream
```

**WebSocket** — must implement manually:
```js
function connect() {
  const ws = new WebSocket(url);
  ws.onclose = () => {
    setTimeout(connect, Math.min(1000 * 2 ** retries++, 30000)); // exponential backoff
  };
}
```

Always use **exponential backoff with jitter** to avoid reconnect storms:
```
retry 1: 1s + rand(0, 0.5s)
retry 2: 2s + rand(0, 1s)
retry 3: 4s + rand(0, 2s)
...capped at 30s
```

### Message Ordering

Network packets can arrive out of order. For commentary or collaborative edits, order matters.

**Solution: sequence numbers + client buffer**

```js
// Server assigns monotonically increasing sequence
{ seq: 101, data: "Ball 32.3: Dot ball" }
{ seq: 102, data: "Ball 32.4: SIX!" }
{ seq: 103, data: "Ball 32.5: Wicket!" }

// Client buffers and renders in order
const buffer = new Map();
let nextExpected = 102;

function onEvent(event) {
  buffer.set(event.seq, event);
  while (buffer.has(nextExpected)) {
    render(buffer.get(nextExpected));
    buffer.delete(nextExpected);
    nextExpected++;
  }
}
```

### Server Failures

When an SSE server dies:
1. All TCP connections on that server are forcibly closed by the OS
2. Clients detect the disconnect and begin reconnecting
3. Load balancer routes reconnects to healthy servers
4. New server receives `Last-Event-ID` and replays missed events from Kafka
5. New server subscribes to Pub/Sub channels for those users

---

## Networking Fundamentals

Understanding why these protocols exist requires knowing the layers they build on.

```
OSI Layer    Protocol          What it does
──────────────────────────────────────────────────────
Layer 7      HTTP, SSE,        Application logic, data formats
             WebSocket, WebRTC
Layer 4      TCP, UDP          End-to-end delivery
Layer 3      IP                Routing between networks
Layer 1-2    Ethernet, WiFi    Physical transmission
```

**TCP** (used by HTTP, SSE, WebSocket):
- Connection-oriented (3-way handshake before data)
- Guaranteed delivery, guaranteed order
- Cost: overhead to establish and maintain connection

**UDP** (used by WebRTC media):
- Connectionless, no handshake
- No delivery or order guarantees
- Benefit: lowest latency (no retransmit wait)

**HTTP/2 advantage for SSE:**
- HTTP/1.1 limits 6 connections per domain → SSE uses one per tab
- HTTP/2 multiplexes many streams over one TCP connection
- Multiple SSE streams from same domain share one connection

---

## Real-World Examples

### Cricbuzz (Live Cricket Scores)

```
Ball bowled
    │
    ▼
Score Ingestor Service
    │
    ├──PUBLISH match:IND-AUS──▶ Redis Pub/Sub
    │                               │
    │                    ┌──────────┴──────────┐
    │                    ▼                     ▼
    │              SSE Server 1          SSE Server 2
    │              (subscribed)          (subscribed)
    │                    │                     │
    │              push to local         push to local
    │              connections           connections
    │
    └──write──▶ DB (source of truth)
               └──write──▶ Redis cache (scorecard, short TTL)
```

- **Hop 1:** SSE (one-directional, browser auto-reconnect)
- **Hop 2:** Redis Pub/Sub
- **Reconnect replay:** Redis buffer or Kafka for last N events
- **Scorecard page:** Redis cache with short TTL (10-15s)

---

### Google Docs (Collaborative Editing)

```
User A types "hello"
    │
    ▼
WebSocket Server
    │
    ├──apply Operational Transform (OT)──▶ resolve conflicts
    │
    ├──PUBLISH doc:xyz-changes──▶ Kafka
    │                               │
    │                    ┌──────────┴──────────┐
    │                    ▼                     ▼
    │              WS Server 1           WS Server 2
    │              (User B)              (User C)
    │
    └──write──▶ DB (document state)
```

- **Hop 1:** WebSocket (bidirectional — users also send keystrokes)
- **Hop 2:** Kafka (need replay, ordering guarantees)
- **Conflict resolution:** Operational Transform or CRDT

---

### WhatsApp (Message Delivery)

```
User A sends message
    │
    ▼
Chat Server (User A's WS connection)
    │
    ├──store──▶ Message DB
    │
    └──lookup──▶ "Which server holds User B's connection?"
                 Redis: conn:UserB → Server 3
                    │
                    ▼
              Server 3 ──push──▶ User B (WebSocket)
                    │
              Server 3 ──ACK──▶ Chat Server ──ACK──▶ User A (✓✓)
```

- **Hop 1:** WebSocket
- **Hop 2:** Direct server-to-server routing via connection registry in Redis
- **Offline users:** Message stored in DB, delivered via push notification (FCM/APNs), pulled on reconnect

---

## Interview Cheat Sheet

### When to Use Which Protocol

| Use Case | Protocol | Reason |
|----------|----------|--------|
| Live scores, stock tickers | SSE | One-way, built-in reconnect |
| Push notifications feed | SSE | Server-to-client only |
| Chat application | WebSocket | Bidirectional |
| Collaborative document editing | WebSocket | Bidirectional, low latency |
| Multiplayer game | WebSocket | Bidirectional, sub-100ms |
| Video/audio calling | WebRTC | P2P, media optimized |
| Admin dashboard refresh | Simple Polling | Infrequent, simple |
| Basic alerts, no SSE support | Long Polling | Fallback |

### When to Use Which Hop 2 Strategy

| Scenario | Strategy |
|----------|----------|
| Simple, low scale | Polling from source |
| Predictable user-to-server mapping | Consistent Hashing |
| High scale, many servers | Pub/Sub (Redis) |
| Need replay / audit trail | Pub/Sub (Kafka) |
| Direct user-to-user routing | Connection registry in Redis |

### Common Deep Dive Questions

**"How do you handle reconnection?"**
- SSE: `Last-Event-ID` header sent automatically, server replays from Kafka/buffer
- WebSocket: exponential backoff with jitter, re-subscribe on reconnect

**"What if one user has 50M followers who all need the same update?"**
- Fan-out via Kafka with 100s of consumer workers
- Each worker handles a shard of follower SSE servers
- Pre-compute fan-out lists for celebrities (Twitter's approach)

**"How do you maintain message ordering?"**
- Assign monotonically increasing sequence numbers at source
- Client buffers out-of-order events and renders only contiguous sequences
- Single Kafka partition per entity (match/chat room) guarantees order

**"How do you scale SSE/WebSocket servers?"**
- Horizontal scale with Pub/Sub — servers are stateless w.r.t. routing
- Each server only holds connection objects in RAM
- Load balancer distributes new connections (sticky sessions not required with Pub/Sub)

**"What happens during a traffic spike?"**
- Jittered cache TTL prevents simultaneous expiry
- Cache locking prevents thundering herd on DB
- Pub/Sub absorbs the burst before it hits SSE servers
- Rate limit notification delivery (stagger FCM batches)

---

### Architecture Decision Flowchart

```
Need real-time updates?
        │
        ▼
Does client need to SEND data too?
   YES → WebSocket
   NO  → SSE (preferred) or Long Polling (fallback)
        │
        ▼
Is it media (video/audio)?
   YES → WebRTC
   NO  → stay with WebSocket/SSE
        │
        ▼
For Hop 2: how many servers?
   1 server     → direct publish
   Few servers  → Consistent Hashing
   Many servers → Pub/Sub (Redis or Kafka)
        │
        ▼
Need event replay on reconnect?
   YES → Kafka (seekable offsets)
   NO  → Redis Pub/Sub (ephemeral, simpler)
```

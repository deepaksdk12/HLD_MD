# Uber (Ride Sharing) — System Design Interview Notes

## Understanding the Problem

Uber's CRUD surface is small: a rider asks for a ride, a driver accepts it, a trip happens, a fare gets charged. The hard part is everything hiding inside "a driver accepts it" — out of potentially thousands of drivers near a rider, the system has to pick one, offer it to them, and guarantee that exact same driver can't simultaneously be offered (or committed) to a different rider a street away. Layer on top of that a firehose of GPS pings — every online driver reporting their position every few seconds, nonstop, whether or not anyone is requesting a ride right now — and you have a system where the write volume of "where is everyone" dwarfs the write volume of "who's riding with whom" by orders of magnitude. The interesting design problems are: fast nearby-driver lookup, race-free matching, and surviving the location-ping flood without that flood ever being allowed to slow down an actual ride request.

### Functional Requirements

- Riders can get a fare estimate (ETA + price) for a given pickup and dropoff.
- Riders can request a ride based on that estimate, and the system matches the request to a nearby available driver in real time.
- Riders can cancel a ride request or an in-progress trip.
- Drivers can go online/offline, and their live location is tracked while online.
- Drivers can accept or deny an incoming ride request, then navigate to pickup and on to dropoff.
- Riders and drivers can see each other's live location during an active trip.
- The system calculates a fare and charges the rider when a trip completes.

We're explicitly **not** building route/turn-by-turn navigation or ETA computation from scratch (a third-party maps provider handles that), not building the surge-pricing algorithm itself (we'll treat "the fare" as a black-box function of distance/time/demand), not building driver onboarding/background-check/vehicle-verification flows, not building multiple vehicle tiers (UberX only), rider/driver ratings, or scheduling a ride in advance, not building **ride pooling** (multiple unrelated riders sharing one trip — a genuinely different matching and fare-splitting problem), and — distinct from the ETA black-box above — not building precise post-trip **route reconstruction** (map-matching a noisy stream of GPS pings back onto the actual roads driven, for exact-distance billing or analytics). That last one looks related to the routing we're already excluding but is a genuinely separate, hard problem (probabilistic road-matching, typically a Hidden Markov Model over GPS signal/road-transition probabilities, solved with the Viterbi algorithm) — worth naming explicitly as a real extension rather than silently waving it into the Maps-API black box, but out of scope for this design.

### Non-Functional Requirements

Like Ticketmaster, these split by code path, and every deep dive below maps back to one of them:

- **The match/dispatch path is consistency over availability.** A driver must never be committed to two trips at once — that's a driver showing up to the wrong pickup, or not showing up at all, which is a worse failure than briefly rejecting a request and retrying.
- **The location-ping path is availability over consistency, and must absorb enormous write volume.** A driver's dot on the map being one or two seconds stale is invisible to the user; refusing to accept that ping because some downstream index is momentarily behind is not acceptable — pings must never back up or get dropped under load.
- **Low matching latency, quantified** — a rider should get a driver or a clear "no drivers available" failure in **under 1 minute**, not an indefinite wait.
- **High availability for the request → match → trip path** — a failed ride request is lost revenue and a user who opens a competitor's app.
- **Low latency even under degraded network conditions** — the app has to stay responsive on the patchy cellular coverage that's common exactly where a lot of pickups happen (parking garages, stadiums, rural areas); location pings and status updates need to tolerate retries and brief disconnects without surfacing errors to the user or losing the rider's place in a match.
- **Handle high throughput surges, quantified** — unlike steady DAU-driven growth, a popular event ending (a stadium concert, New Year's Eve) can produce **on the order of hundreds of thousands of ride requests in a single region within a few minutes** — a sudden, correlated spike, not gradual load growth.

### Back-of-the-Envelope

Only carrying the scale assumptions needed before the design trace. The arithmetic shows up later at the exact decision point it justifies, instead of front-loading every calculation here.

**Assumptions:**

- ~5M drivers online at peak globally, each pinging location every 4s.
- ~1M ride requests at peak hour across all cities.
- Average trip lasts ~15 minutes.
- "Nearby drivers" search radius covers roughly 100-500 candidate drivers per request in a dense city.
- A single event can create ~100K-300K ride requests in one region within ~5-10 minutes.

**Initial takeaways:**

- Location writes dominate every other write path, so live location cannot be treated like ordinary durable trip data.
- Ride requests are lower volume than location pings but burst hard by region, so dispatch needs queueing/backpressure once we scale.
- Trip records are durable, relational state and are still a good fit for PostgreSQL; live location is ephemeral and gets handled separately.
- The detailed sizing for location writes, trip storage, and event surges appears in the deep dive or DB-selection section that uses each number.

---

## Core Entities

- **Rider** — the person requesting a trip.
- **Driver** — the person providing it; has a current `Status` (offline/online/matched/on_trip) that the matching algorithm depends on being correct.
- **Vehicle** — the car a driver is driving (make/model/plate), looked up for the rider's benefit at match time.
- **Trip** — one ride from fare estimate through completion: who, pickup/dropoff, status, fare.
- **Location** — a driver's current position. Deliberately *not* modeled as a row-per-ping in the durable store — see Deep Dive 1 for why this lives in an ephemeral index instead of a table.

---

## API Design

```
POST /riders/{riderId}/fare-estimates    -> { tripId, fare, etaSeconds }
body: { pickupLocation, dropoffLocation }

POST /trips/{tripId}/request             -> RideRequest { tripId, status }
                                             (requests a ride against an
                                              existing fare-estimate's tripId)

POST /drivers/{driverId}/status          -> 200
body: { status: 'online' | 'offline' }

PATCH /drivers/{driverId}/location       -> 200
body: { lat, lng, heading, timestamp }

POST /trips/{tripId}/respond             -> Trip      (driver accepts or denies the offer)
body: { accept: boolean }

POST /trips/{tripId}/start               -> Trip      (driver has picked up rider)

POST /trips/{tripId}/complete            -> Trip & Receipt

POST /trips/{tripId}/cancel              -> Trip      (rider-initiated)

GET  /trips/{tripId}                     -> Trip       (poll/fallback for live state)

GET  /riders/{riderId}/trips             -> Trip[]
```

All endpoints are versioned under `/api/v1/...` so the request/response shape can evolve without breaking existing clients, and every request carries its caller's identity via an auth token (JWT or session cookie) resolved server-side from the request header — never a `riderId`/`driverId` field accepted from the body, for the same reason Ticketmaster's API avoided it: a client-supplied ID would let any authenticated user act on someone else's trip.

Notice the fare estimate and the ride request are **two calls, not one** — `POST /fare-estimates` creates a `Trip` row in a `fare_estimated` state purely to get a price/ETA in front of the rider, and a separate `POST /trips/{tripId}/request` is what actually kicks off dispatch. That split means a rider who looks at a price and closes the app costs us nothing beyond one row (useful for analytics on abandoned quotes, too), and it means the *expensive* part — searching for and holding a driver — only ever starts once a rider has committed.

Also notice `respond` is explicit about `accept: boolean`, not just an "accept" endpoint — a driver actively declining should free up the offer and move dispatch to the next candidate immediately, the same as a timeout would, without waiting out the full offer window. `cancel` is similarly explicit and separate from `respond`'s decline — one is the driver walking away from an *offer*, the other is the rider walking away from their own *request*, and they trigger different cleanup (see Deep Dive 8).

`PATCH /drivers/{driverId}/location` is its own tiny, high-frequency endpoint, separate from everything else a driver's app does — that's deliberate. It's called every few seconds for every online driver whether or not a trip is happening, so it has to be cheap enough to write without touching the same database that holds trip and payment records (more in Deep Dive 1).

Also notice matching isn't a single synchronous call that returns "Trip, with driver attached." `POST /trips/{tripId}/request` kicks off a dispatch process and returns immediately; the client then watches (via the live-tracking channel from Deep Dive 4, with `GET /trips/{tripId}` as a polling fallback) — because the actual match (find a driver, offer it, wait for accept, or retry the next-nearest driver) can take up to a minute and shouldn't hold the request thread open.

---

## High-Level Design

Let's trace a ride end-to-end and let the components fall out of the trace.

**A driver goes online and starts moving.** Their app calls `POST /drivers/{id}/status` once, then `PATCH /drivers/{id}/location` continuously. In this simple version there's a single **Trip Service** instance handling everything — no load balancer yet, since there's nothing to balance across — and it writes the driver's current status and location straight into the **Rider/Driver/Trip DB** (PostgreSQL).

**A rider asks for a fare estimate.** `POST /fare-estimates` hits the Trip Service, which calls a third-party Maps API for an ETA/route given the pickup and dropoff, derives a price from it (a black box per our out-of-scope note), and writes a new `Trip` row in a `fare_estimated` state — cheap, synchronous, no driver search yet.

**The rider requests the ride.** `POST /trips/{tripId}/request` hits the Trip Service, which needs to find a nearby `online` driver. In this basic version that's a query against the same Postgres table the location writes just landed in (e.g. drivers within a lat/lng bounding box). The Trip Service picks the nearest available driver, flips that driver's status, and offers the trip.

**The driver accepts or denies, drives, and completes the trip.** Each state transition (`respond`, `start`, `complete`) is a write to the same `Trip` row, ending with a fare calculation and a charge against the rider's stored payment method.

Putting it together: **Client** (rider or driver app) talks directly to the **Trip Service**; the Trip Service is the only thing that reads and writes the **Rider/Driver/Trip DB**; a third-party Maps API (for ETA/routing) and a Payment Gateway sit outside our system entirely.

**Database Selection: PostgreSQL vs. NoSQL (e.g. DynamoDB) vs. a geospatial store.** There are really two different access patterns fighting for the same database here, and that's the whole justification.

**Bad: one PostgreSQL instance holding both trip records *and* every location ping as rows.** The match/trip path wants exactly what Postgres is good at — multi-row transactions (atomically flip a driver's status and create a trip row together) and simple relational lookups. But the location sizing in Deep Dive 1 works out to ~1.25M writes/sec at global peak, which would overwhelm any single relational instance's write throughput almost immediately — and durability/ACID guarantees are wasted on data that's stale and irrelevant two seconds after it's written.

**Good: DynamoDB (or similar) for everything, partitioned by DriverId/TripId.** Auto-sharding absorbs the location-write volume fine. But a trip's state-transition logic (don't let two concurrent `accept` calls both succeed on the same `Trip`, don't dispatch a driver who's mid-transition to `matched`) wants the same single-row atomicity Ticketmaster's seat-hold needed — achievable with conditional writes, but every join-like lookup (a trip's rider + driver + vehicle details) now means multiple round-trips or denormalization, for data that's genuinely relational and doesn't actually need to scale past what one well-sharded relational cluster handles.

The storage math that matters here is trip history, not location: 1M rides/hour × 24h ≈ 24M trips/day; at roughly 1KB/trip, that's ~24GB/day, or ~17TB over two years. That's big, but it is comfortably partitionable by region/time in PostgreSQL and not by itself a reason to give up relational joins and transactions.

**Best (chosen): split the two concerns.** Trip, Rider, Driver profile, and Vehicle records live in **PostgreSQL** — relational, transactional, and sized well within what a sharded-by-region Postgres cluster handles. Live driver *location* lives in a separate, purpose-built in-memory geospatial index (Redis GEO or an H3-based store) that's built for exactly this write volume and exactly this query shape ("who's near this point right now") — detailed in Deep Dive 1. Postgres's `Driver.Status` column, not the location index, remains the source of truth for whether a driver is assignable.

**Trade-off:** we're running two stores instead of one, and a driver's "current location" now has no durability guarantee at all — if the in-memory index is lost, we lose live positions (not trip history, not driver identity, not fare records). We accept that because location data is inherently ephemeral and self-healing: the next ping a few seconds later repopulates it, and no correctness property of the system depends on yesterday's GPS ping being retrievable.

This satisfies the functional requirements with the simplest shape that works — but it says nothing about *how* a nearby-driver query runs fast at the scale above, what stops two riders from being matched to the same driver, how the location-ping firehose avoids becoming the bottleneck, how either party sees the other move in real time, how a correlated surge of ride requests avoids overwhelming Dispatch, how the whole system stays available beyond one region, or what happens when a rider changes their mind partway through. That's what the deep dives are for.

---

## Deep Dive 1: How do we find nearby drivers fast, and absorb 1.25M location writes/sec?

**The problem:** "find available drivers within 2km of this point" is not a query a row-per-driver relational table answers efficiently at any real scale — a naive `WHERE lat BETWEEN ... AND lng BETWEEN ...` scan degrades badly as the number of concurrent drivers grows, and that's before accounting for the write side: every one of those drivers is overwriting their row every few seconds.

This is where the location-write math actually matters: 5M online drivers / 4s per ping ≈ **1.25M location writes/sec** at global peak. That number is the reason live location cannot be a normal durable row update, and why every option below is judged primarily by write throughput and nearby-radius lookup latency.

**Bad: linear scan + Euclidean distance over all drivers in a city.** Correct, but O(n) per request against a growing driver pool, and it reads from the exact table that's being hammered with ~1.25M writes/sec — reads and writes contending on the same hot rows.

**Also rejected: a quad-tree.** A quad-tree (recursively split the map into four quadrants, subdividing further wherever driver density is high) handles uneven density well — exactly what a dense downtown vs. a sparse suburb needs — and is a legitimate answer in other proximity-search problems (Yelp-style "find nearby points of interest," where writes are rare). It's the wrong fit specifically *here* because the tree structure itself needs rebalancing as points move, and our driver positions are being rewritten by the full location-ping firehose, not occasionally edited — paying a tree-rebalance cost on close to every write is the opposite of what the location-write math demands.

**Good: a relational table with a geohash column and a prefix index.** Encode each driver's lat/lng into a geohash string; "nearby" becomes a prefix match, which an index can serve reasonably well. This is a real improvement in query shape, but it's still a relational table absorbing the full location-write firehose, and Postgres has no native notion of "radius" — prefix matching at a geohash cell boundary also has an edge-case bug where two points a few meters apart end up in totally different prefixes.

**Best (chosen): an in-memory geospatial index — Redis GEO (or Uber's own H3 hexagonal grid) — fully separate from the durable Trip DB.** `GEOADD drivers <lng> <lat> driverId` on every ping; `GEOSEARCH drivers FROMLONLAT ... BYRADIUS 2 km` to find candidates. This is in-memory, so it absorbs the write volume Postgres never could, and its native radius query sidesteps the geohash-prefix edge case entirely. H3 (hexagonal cells at multiple resolutions) is the same idea with better distance-distortion properties near cell boundaries and is what Uber actually built for this reason — either works for the interview-level argument.

**What if the initial radius comes up empty, or with too few candidates?** A fixed 2km search isn't always right — a dense downtown core might have 500 candidates in 500m, while a rural pickup might have none within 5km. The Dispatch Service's search is therefore iterative: start at a small radius (or, for a cell-based index, the driver's own geohash/H3 cell), and if the candidate count comes back under some threshold, expand to the next ring of neighboring cells and search again, repeating until either enough candidates are found or a radius cap is hit. Hitting that cap with zero candidates is exactly the "no drivers available" failure branch the matching NFR (match or fail in under a minute) accounts for — a real answer, not an open-ended wait.

**A search radius doesn't respect partition boundaries.** Once the geo-index itself is sharded by region (Deep Dive 7), a pickup point near a shard's edge has a real problem: its 2km circle can include cells owned by a neighboring partition. The node that owns the requesting pickup point's cell can't just ignore those — it fans the same radius query out as an internal RPC to the neighboring partition(s) that own the overlapping cells, and merges the returned candidate lists before ranking. This is a normal cost of sharding anything by geography — you occasionally pay a cross-shard hop for queries that straddle a boundary — and it's cheap here specifically because it's a single extra RPC hop to one or two known-neighbor partitions, not a broadcast to every shard in the system.

**Nearest-by-straight-line isn't the same as fastest-to-arrive.** `GEOSEARCH`'s radius is Euclidean distance on a sphere — it has no idea about one-way streets, a river with one bridge crossing, or a highway that makes a "farther" driver actually faster. So the geo-index search is a cheap first filter, not the final ranking: Dispatch takes the top-K candidates by raw distance (K small — a handful, not hundreds) and calls the same Maps API already used for the rider's fare-estimate ETA to get each candidate's actual driving ETA, then offers in that order instead of raw-distance order. This keeps the expensive routing calculation bounded (K calls, not one per candidate in the radius) while fixing the case where the "closest" driver on a map is not the one who'll actually arrive first.

**Trade-off:** this index has no durability and no transactional guarantee — a driver's last-known position can vanish on a cache restart, and it is deliberately *not* the source of truth for whether a driver is assignable (Postgres's `Driver.Status`, per Deep Dive 2, is). We're trading durability for the raw write throughput and query latency nothing else here provides, which is the right trade since position data is self-healing within seconds.

**A cheaper fix sitting next to the expensive one: do we need 1.25M writes/sec at all?** Everything above assumes every online driver pings every 4 seconds unconditionally — that number is a choice, not a law. The client can adapt its own ping interval: a driver who's `offline` or has been stationary for a while needs far less frequent updates than one actively driving with an offer pending; a driver sitting idle in a low-demand area doesn't need the same cadence as one in a dense downtown core during rush hour. This is a pure win — it reduces load on Deep Dive 3's ingestion pipeline for free — but it's a mitigation on top of the geo-index decision above, not a substitute for it: even a 5-10x reduction still leaves a write volume no durable relational store should take directly.

**What happens when a geo-index node itself dies?** Since the index is deliberately non-durable (the trade-off above), losing a node loses every driver position it held — but that's recoverable within seconds, not catastrophic, because every online driver is about to re-send their position on the next ping cycle regardless. The operational concern is routing: a consensus mechanism (Redis Sentinel, or a Raft-based coordinator) detects the dead node and promotes a standby replica to take over that shard, and the Dispatch Service's view of "which node owns this geohash/H3 cell range" needs to repoint at the new replica. The one thing worth guarding against is a **thundering herd on failover** — if every driver whose pings were routed to the dead node reconnects to the newly-promoted replica at the exact same instant, that's a connection spike against a node that just came up cold; a small randomized jitter on the client's reconnect delay (same idea as Deep Dive 7's partition-handoff reconnect, below) smooths that out cheaply.

---

## Deep Dive 2: How do we stop two riders from being matched to the same driver?

**The problem:** this is actually two separate races, and conflating them is the easy mistake to make.

**Race A — the same ride offering itself to multiple drivers at once.** This one is easy: a single `Trip`'s dispatch loop (search → offer → wait → retry-next-candidate) runs sequentially inside one Dispatch Service instance handling that specific request. There's no concurrency to coordinate — it's a `while (no match) { offer next candidate }` loop on one thread, nothing more.

**Race B — the same driver being offered two different trips at once, from two different Dispatch Service instances.** This is the hard one. Finding candidate drivers (Deep Dive 1) is a read against an eventually-consistent, in-memory index — it can hand back a driver who was already claimed microseconds ago by a *different* request being handled on a *different* instance. If "assign driver" is just "write `matched` to their status," two concurrent requests that both read the same driver as available can both try to commit, and one driver ends up double-booked — or, just as bad, gets a second ride offer shoved in front of them mid-trip.

**Bad: check-then-write.** `SELECT status FROM drivers WHERE id=?`, then `UPDATE ... SET status='matched'` if it read `available`. Classic TOCTOU race — two requests can both pass the check before either `UPDATE` commits.

**Bad: a periodic cleanup job for stuck offers.** Suppose we do lock a driver by writing `status='offered'` — something still has to unlock a driver whose offer timed out without a response. A cron job that scans for `status='offered'` rows older than the timeout and resets them works, but only at the job's granularity: a job running once a minute means a driver who should free up after a 10-second timeout can stay locked for up to 50 extra seconds, actively hurting the "match in under a minute" NFR during exactly the high-load moments it matters most.

**Good: pessimistic row lock (`SELECT ... FOR UPDATE`).** Correct, but holds a database lock for however long the whole offer/accept round-trip takes — and that round-trip includes *pushing a notification to the driver's phone and waiting (up to several seconds) for them to tap Accept*. Locking a row in Postgres for multiple seconds of network-and-human latency is exactly the kind of long-held lock that creates its own contention problems if several ride requests land near the same cluster of drivers.

**Best (chosen): atomic conditional write + a self-expiring lock, decoupled from the human accept step.** `UPDATE drivers SET status='offered', offered_trip_id=? WHERE driver_id=? AND status='available'` — zero rows affected means we lost the race (or more precisely, raced correctly and someone else's write landed first), and we simply move on to the next-nearest candidate from Deep Dive 1's search. In parallel, a short-TTL key (`SETNX driver_lock:{id}, EX 10` in Redis, or an item with a native TTL attribute in DynamoDB) gives every Dispatch Service instance a fast, consistent "is this driver currently locked" check before it even attempts the conditional write — no lock is held while we wait on the driver's phone, and an unanswered offer self-expires in exactly 10 seconds, not "whenever the next cleanup job happens to run."

**A late response is a stale response, not a new race.** Network latency on a driver's accept response is unbounded — it's entirely possible a driver taps Accept on an offer that the Dispatch Service has *already timed out and reassigned to someone else*. This isn't a new problem requiring new machinery: `respond`'s handler checks the caller's `driverId` against the `Trip`'s current `offered_trip_id`/assigned driver before accepting anything, the same fencing-token check Ticketmaster's `HeldByUserId` used. By the time a late response arrives, the timeout reclaim has already overwritten that driver's `offered_trip_id` (to null, or to a different trip entirely) — so the late accept finds a mismatch and is rejected, not silently double-accepted. No part of this depends on *when* the response happens to arrive; it depends only on what the row currently says.

**Trade-off:** this is first-successful-write-wins dispatch, not a guarantee that the geographically closest driver always gets the trip — a driver slightly further away whose conditional write lands first can win over one slightly closer whose write arrives moments later. We accept that: optimizing for "never double-book a driver" and "don't hold locks across human response time" matters more than shaving a few hundred meters off the average match.

---

## Deep Dive 3: How does the location-ping ingestion pipeline avoid becoming the bottleneck?

**The problem:** Deep Dive 1 established *where* location data lives (an in-memory geospatial index), but 1.25M writes/sec landing directly and synchronously on that index from driver apps still couples driver app latency to that index's instantaneous health — a brief blip in the geo-index tier would otherwise propagate straight back to every driver's app as failed pings.

**The chosen approach:** driver apps publish location pings to **Kafka** instead of writing to the geo-index directly; a pool of consumers reads off Kafka and applies `GEOADD`/H3-cell updates to the index asynchronously. This is the standard "decouple a write-heavy producer from a downstream store's momentary capacity" move — if the geo-index tier needs to scale out or briefly hiccups, Kafka absorbs the backlog instead of the driver's `PATCH /location` call timing out.

**Trade-off:** a driver's position in the index now lags their actual location by however long it takes to drain the Kafka partition — normally milliseconds, but it can grow under a consumer-side incident. That's an acceptable trade given the NFR (availability over consistency on this path): a dispatch decision made against a position that's a few hundred milliseconds stale is still a fine decision; a location-ping endpoint that falls over under load is not.

---

## Deep Dive 4: How do rider and driver see each other move in real time during a trip?

**The problem:** once a trip is accepted, both the rider and driver apps need to render the other party's live position — the rider watching their driver approach, the driver seeing the rider's pin. Polling `GET /trips/{id}` on a fixed interval is the naive answer and has the same bad trade-off Ticketmaster's seat-map staleness deep dive hit: too slow and the dot jumps; too fast and you're spamming the Trip Service from every active trip's two participants.

**Bad: fixed-interval polling.** Simple, no new infrastructure, but tuning the interval is a lose-lose — accurate tracking wants sub-second updates, and sub-second polling from every active trip's rider *and* driver multiplies request volume for no benefit over a push model.

**Also considered: Server-Sent Events, and QUIC.** SSE gives one-way server-to-client push over plain HTTP — simpler than a WebSocket, but the driver side of this problem needs to *send* location pings continuously anyway (Deep Dive 3), so a connection type that can't carry both directions doesn't remove any infrastructure, it just adds a second channel. QUIC (HTTP/3's transport) is what Uber's own production stack actually uses for this problem — faster connection setup and better behavior under lossy mobile networks than a WebSocket's TCP handshake — but it's a newer, less broadly-understood protocol with a smaller ecosystem of off-the-shelf tooling; for an interview-level design, WebSocket demonstrates the same bidirectional real-time pattern with a far smaller "why would I even use this" explanation burden.

**Best (chosen): a persistent one-way stream (WebSocket or SSE) from a dedicated Location/Notification Service.** Once a trip is `accepted`, both parties open a connection to this service; the already-decoupled Kafka pipeline from Deep Dive 3 is the natural fan-out point — a consumer on that same location-event stream forwards a driver's position straight to any rider (and vice versa for trip-status changes) currently subscribed to that `tripId`, instead of a second write path being invented just for this.

**The connection handshake, concretely:** the client opens a `wss://` connection to the Location/Notification Service and, immediately after the socket is established, sends its auth token (the same JWT used for REST calls) as the first message — the service validates it and maps the now-authenticated connection to that user's ID, then the client sends a `subscribe: { tripId }` message to join that specific trip's channel. This matters for two reasons: the token is sent *after* the connection opens, not as a URL query parameter (which would leak into server logs and proxies), and mapping connection → user → tripId is what lets the fan-out consumer above know exactly which open sockets to push a given trip's updates to, instead of broadcasting to everyone.

**Scaling the connection tier itself:** an open WebSocket is pinned to whichever Location/Notification Service instance accepted it — a load balancer in front of this tier needs sticky sessions (or a connection-aware routing layer) so a reconnect or a message destined for that user lands back on the same instance holding their socket state, not a random one that's never heard of them. A periodic heartbeat (client ping / server pong on an interval) is what actually detects a half-dead connection — a mobile client that lost signal without a clean TCP close — so the server can free that connection's resources instead of holding it open indefinitely.

**Trade-off:** an open connection per active trip-participant is real server-side state, unlike stateless polling — but the number of *concurrent trips* is a much smaller number than the number of *online drivers* (per the Back-of-the-Envelope match-rate estimate), so this is cheap relative to the location-ingestion problem it reuses infrastructure from.

---

## Deep Dive 5: How do we calculate and charge the fare without blocking trip completion?

**The problem:** `POST /trips/{id}/complete` needs to compute a fare (distance, time, surge multiplier — treated as a black box per our out-of-scope note) and charge the rider's card. If that charge happens synchronously inside the `complete` call, a slow or down payment gateway makes *ending a trip* fail, which is a bad place for a third-party dependency to sit in the critical path.

**The chosen approach:** `complete` writes the `Trip` as `completed` with a `PaymentStatus: pending` immediately — the driver and rider both see the trip end right away — and publishes a `trip completed` event to **Kafka**. A separate consumer computes the final fare and calls the Payment Gateway asynchronously, updating `PaymentStatus` to `charged` or `failed` once it resolves. This is the same "don't let a downstream side-effect block the user-visible path" pattern as Ticketmaster's notification deep dive.

As with any client-retriable financial call, the charge itself needs an idempotency key (generated once, at trip-complete time, stored on the `Trip` row) so a retried charge attempt — from a consumer restart, a network blip to the gateway — never double-charges the rider.

**Trade-off:** a rider's receipt and final charge are now eventually consistent with trip completion, not instant — acceptable, since nothing about the trip itself (both parties already know it ended) depends on the payment settling within the same second.

---

## Deep Dive 6: How do we survive a surge of ride *requests* (not just location pings)?

**The problem:** Deep Dive 3 solved the steady, enormous firehose of location pings — but a stadium concert letting out creates a different shape of load entirely. This is where the request-surge math matters: steady global peak is ~1M requests/hour, or **~280 requests/sec average** globally; even a 5-10x regional peak is roughly **~1,500-2,800 requests/sec**. A single popular event can instead put **~100K-300K ride requests into one region within ~5-10 minutes**, a burst roughly 100x the steady-state regional rate. All of those requests want Dispatch to run a relatively expensive, multi-second search-offer-wait loop at the same moment. Unlike location pings, these requests can't simply be dropped or indefinitely delayed — each one is a user actively waiting for the "under 1 minute" matching NFR to hold.

**The chosen approach:** ride requests land on a **Ride Request Queue** (partitioned by region) before the Dispatch Service ever touches them, rather than hitting Dispatch directly off the load balancer. Dispatch Service instances pull from their region's partition at whatever rate they can sustainably process, which turns an uncontrolled, correlated spike into backend-controlled, steady consumption — the same shape as Ticketmaster's Waiting Room, but here the "queue" is invisible to the end user (a few extra seconds of "finding you a driver…") rather than an explicit visible position number, since Uber's wait is bounded by the 1-minute NFR rather than a multi-hour on-sale queue.

**Why partition by region, specifically, and not just add more queue consumers:** a plain FIFO queue lets a hard-to-match request (a rider out in a sparse suburb with few nearby drivers) sit at the head of line and block easier requests behind it (a rider in dense downtown with drivers everywhere) from being picked up promptly — those two requests have nothing to do with each other and shouldn't share ordering. Partitioning by region means a sparse, slow-to-match region's backlog never delays an unrelated, easy-to-match region's throughput.

**A second benefit that falls out for free:** if a Dispatch Service instance crashes mid-search (it can — the search-offer-wait loop takes up to the full matching timeout), the request was never popped-and-acknowledged off the queue, so another instance simply picks it up and retries the search. Without the queue, a crashed instance mid-match would silently lose that rider's request with no retry path.

**Trade-off:** every ride request now takes an extra hop (queue write + consumer pull) versus a direct synchronous call, adding a small fixed latency even at normal, non-surge load. We accept that constant small cost because the alternative — Dispatch Service instances falling over under a correlated spike — is a complete NFR violation (no match, no "sorry" message, just a timeout), while the queue's added latency is invisible against the 1-minute matching budget.

---

## Deep Dive 7: How do we stay highly available outside the matching path?

**The problem:** our consistency-over-availability NFR is scoped specifically to matching (Deep Dive 2) — everything else (browsing, status updates, trip history) should favor staying up over staying perfectly synchronized. A single-region deployment is a single point of failure for the entire product regardless of how well any individual service scales horizontally within that region.

**The chosen approach:** run the whole stack — Dispatch Service, Trip Service, geo-index, Ride Request Queue, Postgres — **active-active, sharded by region** (northeast, southwest, northwest, etc.), each in its own data center. This falls out naturally from a property we already relied on: a trip never spans two regions (nobody requests a ride that starts in New York and gets dispatched by a California-based Dispatch Service), so routing a request to its region's full stack has no cross-region coordination to get right on the hot path — it's the same locality argument Ticketmaster used to shard `Seat` by `EventId`.

**The one place this isn't clean: a user who travels.** A rider who flies from New York to San Francisco needs their account (profile, payment method, trip history) visible in the California region's Postgres, not just the northeast's. We handle this the ordinary way a profile table handles multi-region read access — a cross-region read replica (or a lightweight global table) for the `Rider`/`Driver` profile tables specifically, which are read far more than written and have no part in the matching hot path's consistency requirements.

**A second place that isn't clean: a driver physically crossing a shard boundary.** A driver's live-tracking WebSocket connection (Deep Dive 4) and their geo-index entry (Deep Dive 1) are both pinned to whatever region/partition owns their current coordinates. When a driver drives across that boundary mid-trip, the connection they're holding is suddenly pointed at the wrong region's stack. The fix is for the losing region to close that connection (rather than silently keep serving stale state) once a location ping lands outside its owned range, which signals the client to reconnect — at which point the Location/Notification Service's connection-routing logic (the same `wss://` → auth → map-to-user flow from Deep Dive 4) simply resolves to the new region. The only extra care needed: a jittered reconnect delay, the same mitigation used for the geo-index failover case in Deep Dive 1 — boundary crossings happen to many drivers near a dense region edge gradually, not all at once, so this is a smaller risk than the failover case, but the same cheap fix applies.

**A harder failure mode: losing an entire region mid-trip, not just one node.** Active-active sharding handles a node dying inside a region (another node in the same region's Postgres/geo-index cluster takes over), but what if the *whole region* goes down while trips are actively in progress there? The naive fallback — route affected clients to a neighboring region — doesn't help by itself, because that region's database has never heard of these trips; it has no `Trip` row, no `Driver.Status`, nothing to resume from. The insight (and what Uber's own infrastructure actually leans on) is that the **driver's own app already holds everything needed to describe the trip's current state** — trip ID, rider ID, pickup/dropoff, current status — because it's been receiving and sending that state over its connection the whole ride. So the failover path isn't "replay the dead region's database," it's "treat the driver app as the source of truth for its own in-flight trip" — the moment the backup region sees a request for a trip ID it has no record of, it accepts the driver app's own last-known state as authoritative and creates the `Trip`/`Driver` rows fresh from it, rather than refusing the request for lack of history.

**Trade-off:** running N full regional stacks instead of one bigger one means more operational surface (N sets of services, N databases, N geo-indexes to keep healthy) and the cross-region profile replication above is itself eventually consistent — a payment method updated seconds ago in one region might not yet be visible from another. We accept both: the alternative (one global region) makes an entire continent's Uber availability depend on one data center's uptime, which directly violates the availability NFR this section exists to satisfy. The driver-app-as-source-of-truth fallback above is itself a trade-off worth naming on its own: it only works because the driver app was already a participant in the trip the whole time — it couldn't resurrect a trip's state from nothing, only corroborate what it personally witnessed.

---

## Deep Dive 8: What happens when a rider cancels?

**The problem:** `POST /trips/{tripId}/cancel` looks like one simple status flip, but what it actually has to do depends entirely on which stage the trip is in, and getting that wrong either strands a driver or double-charges a rider.

**Cancel before any driver is matched (trip still `requested`, sitting in the Ride Request Queue from Deep Dive 6).** The simplest case: mark the `Trip` `cancelled` and, if the Dispatch Service hasn't popped it off the queue yet, it's a no-op when it eventually does (checks `Trip.Status == 'requested'` before doing any work). No driver was ever touched, so there's nothing to clean up on that side.

**Cancel after a driver is offered or has accepted, before pickup.** Here the driver's `Status` is `offered` or `matched`/`on_trip`-pending-pickup — cancelling has to atomically flip the `Trip` to `cancelled` *and* free the driver back to `available` in the same transaction, using the same conditional-write pattern as Deep Dive 2 (`UPDATE drivers SET status='available' WHERE driver_id=? AND offered_trip_id=?`). Skipping this and only updating the `Trip` row would leave a driver permanently stuck off the market for a ride that no longer exists — exactly the kind of two-writes-not-atomic bug Ticketmaster's cancellation deep dive hit with seat release.

**Cancel after the trip is `in_progress`.** Rare (the rider is already in the car) but possible, and it's the one case that interacts with money: if `PaymentStatus` is anything other than `pending`, a late cancellation may still owe a partial fare or cancellation fee — business logic we treat as a black box (same as surge pricing), but the mechanism matters: the cancel handler publishes the same `trip completed`-shaped event to Kafka that Deep Dive 5 already consumes for charging, just tagged with a cancellation reason, so the existing async-charge-with-idempotency-key path handles it rather than a second payment code path being invented.

**Reusing existing machinery instead of building new machinery:** once a driver is freed back to `available`, no new notification mechanism is needed — the same Kafka trip-status-change stream from Deep Dive 4 already fans a `cancelled` status out to both parties over their open WebSocket connections.

**Trade-off:** we don't model a cancellation-fee *schedule* (e.g. free within 2 minutes, $5 after) — that's a pricing/business-logic decision layered on top of this mechanism, not a distributed-systems one, same as the surge-pricing out-of-scope note above.

---

## Data Model

```
Rider                                Driver
- RiderId                            - DriverId
- Name                                - Name
- PaymentMethodToken                  - VehicleId
                                       - Status: offline | online | offered | matched | on_trip
                                       - OfferedTripId        (set during Deep Dive 2's offer window)
                                       - Rating

Vehicle
- VehicleId
- DriverId
- Make / Model / LicensePlate
```

```
Trip
- TripId
- RiderId
- DriverId
- PickupLocation
- DropoffLocation
- Status: fare_estimated | requested | matched | accepted | in_progress | completed | cancelled
- RequestedAt / MatchedAt / AcceptedAt / StartedAt / CompletedAt
- Fare
- PaymentStatus: pending | charged | failed
- PaymentIdempotencyKey
```

`Driver.Status` + `OfferedTripId` is exactly what makes Deep Dive 2's conditional write possible — the DB alone answers "is this driver still free to assign," independent of whatever the (eventually-consistent) geospatial index in Deep Dive 1 thinks. Live `Location` deliberately has **no table here** — it lives only in the Redis/H3 index from Deep Dive 1, because nothing in this data model ever needs to durably query "where was this driver three minutes ago."

### TripStatusHistory (audit log)

```
TripStatusHistory
- TripId
- Status
- ChangedAt
- ChangedBy: 'rider' | 'driver' | 'system'
```

`Trip` itself only ever holds *current* state — overwritten on every transition, per the Database Selection and Deep Dive reasoning above. `TripStatusHistory` is a separate append-only table, one row per transition (`fare_estimated → requested → matched → accepted → in_progress → completed`, or any point a `cancel` cuts in), written alongside each `Trip` update rather than instead of it. This is purely additive — nothing earlier in the design depends on it — but it's what support/dispute resolution actually needs: "the rider says the driver never showed up, what time was each state reached" is a question `Trip`'s current-state-only row can't answer once the trip has moved on.

---

## What We Deliberately Did NOT Add

- **A dedicated API Gateway / Load Balancer in the basic design** — at single-instance scale there's no fleet to route across; it's the natural addition the moment the Trip Service scales horizontally, but adding it before there's more than one instance is an extra hop with no payoff.
- **Our own routing/ETA engine** — a solved problem outside our scope; we call a third-party Maps API and treat it as a black box.
- **The surge-pricing algorithm itself** — a pricing/business-logic problem, not a distributed-systems one; we only modeled that `Fare` is a value computed at completion time.
- **Driver-rider matching fairness/queueing (a FIFO rider queue for scarce drivers)** — mirrors Ticketmaster's fairness note: first-successful-conditional-write wins, not strict ordering, because building a fair cross-rider waitlist adds real complexity for a UX nicety rather than a stated correctness requirement.
- **Multiple vehicle tiers, ratings, and scheduled rides** — real Uber features, but not implied by any stated FR; each would mostly extend the data model (a `VehicleTier` on `Trip`, a `Rating` entity, a future `ScheduledAt` on `Trip`) rather than change the core matching/location architecture above.
- **A separate standalone Driver Service, decomposed from Trip Service** — driver profile/status/availability could be its own service with its own datastore; we kept it inside the Trip Service's Postgres tables because driver status and trip state are read and written together on every match (Deep Dive 2's conditional write touches both), and splitting them would just add a network hop to an operation that wants to be one transaction.
- **A dedicated `Payments`/`Receipts` table, separate from `Trip`** — we track `PaymentStatus`/`PaymentIdempotencyKey` directly on `Trip` since each trip has exactly one charge attempt sequence; a standalone table would matter if we needed multiple historical charge/refund line-items per trip (partial refunds, disputed-then-re-charged amounts), which no stated FR requires.

---

## Potential Follow-Up Deep Dives

- **Multi-driver batched dispatch** — offering a ride to several nearby drivers at once and taking the first accept, instead of walking candidates one at a time, to cut match latency further.
- **Cross-region edge cases for riders near a region boundary** — a rider a few blocks from where our region sharding draws the line (Deep Dive 7) seeing degraded matching if the nearest drivers are technically in the adjacent region's shard.
- **Preferred pickup/drop-off points** — for a location like a large campus or airport with one true entrance, learning (from repeated historical trips) a small set of actual-access points to suggest instead of a raw pin on the map; a real feature, but an ML/data problem layered on top of this design, not implied by any stated FR.
- **Precise route reconstruction (GPS map-matching)** — a genuinely hard, separate sub-problem: modeling noisy GPS pings against the actual road network as a Hidden Markov Model (emission probabilities from GPS-signal-to-road-point likelihood, transition probabilities from road-to-road connectivity), solved with the Viterbi algorithm to find the single most likely path — and, at scale, partitioning that HMM's state the same way the geo-index is partitioned, with a driver crossing a partition boundary needing their in-flight Viterbi state handed off between partitions (e.g. via a Kafka+Flink stateful-stream pipeline) rather than recomputed from scratch.

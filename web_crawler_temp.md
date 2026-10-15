# Web Crawler System Design

## 1. Understanding the Problem

"Fetch pages and follow their links" sounds trivial — download, parse, repeat. The
system design problem shows up once you confront the actual shape of the web: an
effectively unbounded, adversarial graph of URLs where a single misbehaving site can
generate infinite links (a crawler trap), where politeness toward each individual host
matters as much as raw throughput (hammering one small server to death is a bug, not a
feature), and where the frontier of "URLs discovered but not yet fetched" can grow
faster than any fleet can fetch them. The core tension is: you need very high aggregate
throughput across millions of hosts, while being deliberately *slow and polite* against
any single one of them — which rules out a simple shared FIFO queue and forces the
frontier itself to become the central scaling artifact of the whole design.

## 2. Requirements

### Functional Requirements
- Given a set of seed URLs, fetch each page, extract its outbound links, and
  recursively discover and crawl new URLs.
- Store the fetched content (and basic metadata) somewhere downstream systems
  (indexing, archival) can consume it.
- Respect `robots.txt` rules and per-host crawl-delay — never hammer one host.
- Avoid re-crawling URLs already seen, and avoid storing duplicate/near-duplicate
  content under multiple URLs.
- Periodically re-crawl already-seen pages so content doesn't go stale forever.

### Out of Scope
- **Rendering JavaScript-heavy single-page apps.** Headless-browser rendering is a
  much heavier, GPU/CPU-bound pipeline with its own scaling story; this design crawls
  static/server-rendered HTML (and non-HTML content types via extraction plugins) and
  treats JS rendering as a separate, optional downstream stage.
- **Search ranking/indexing itself.** The crawler's job ends at "extracted content,
  durably stored, with its links discovered" — ranking and serving search results is a
  different system that consumes this one's output.
- **On-demand, synchronous crawling of a single page for a live user request.** This
  design is a continuous background discovery/ingestion system, not a
  "fetch-this-URL-right-now-and-return-it" proxy — a user-facing fetch-on-demand
  feature would be a thin, separate, latency-sensitive service.

### Non-functional requirements, and why each one matters
- **Scale to billions of pages, growing continuously.** The frontier (discovered,
  not-yet-fetched URLs) and the fetch/storage pipeline must scale horizontally — no
  single node's disk or CPU can be the ceiling (see BOTE and Deep Dive 3).
- **Politeness is a hard constraint, not an optimization.** Overloading a single host is
  both antisocial and a fast way to get the crawler blocked entirely — this shapes the
  frontier's core data structure (Deep Dive 1) more than any other requirement.
- **Availability over strict consistency.** It's fine if a URL gets crawled slightly
  late, crawled twice, or if the "seen" set has a rare false negative — it is *not* fine
  for one bad host (slow, hanging, redirect loop) to stall the rest of the crawl.
- **Resilience to adversarial/malformed input.** Malformed HTML, infinite redirect
  chains, and deliberately generated infinite URL spaces (crawler traps) must be
  contained per-host, never allowed to consume unbounded fleet resources.
- **Extensibility to new content types.** Adding a new content type (PDF, images) for
  extraction should not require touching the core frontier/fetch loop.
- **Bounded staleness per page, not real-time freshness.** A page being some hours or
  days out of date is fine; recrawl priority should track how often a page actually
  changes rather than treating every page identically (Deep Dive 7).

### Back-of-the-Envelope

Only estimating what feeds a decision below.

**Assumptions:**

- Target: 1B pages crawled/month → ~400 pages/sec sustained average, design for
  ~3x burst (~1,200/sec).
- Average page size: ~500KB (HTML + inline resources counted elsewhere).
- Average out-degree: ~10 outbound links/page discovered per fetch.
- Tens of millions of distinct hosts in the crawl set.

**Estimations:**

Frontier write rate (why a single relational primary can't be the frontier):

- 400 pages/sec fetched × ~10 new links discovered each ≈ 4,000 candidate
  URL-discovery writes/sec sustained, bursting well past that during a fast crawl of a
  large, densely-linked site.
- → a single-primary relational store doing synchronous-commit writes tops out in the
  thousands-to-tens-of-thousands of writes/sec range at best (bound by disk fsync, see
  `postgresql-system-design.md`) — survivable alone, but combined with dedup lookups on
  the same path (Deep Dive 2) and per-host partitioning needs (Deep Dive 1), a single
  primary becomes the throughput ceiling. → favors an auto-sharding NoSQL store for the
  frontier/metadata (DB Selection below).

Storage growth (why content-addressed dedup matters):

- 1B pages/month × ~500KB ≈ 500TB/month of raw content if nothing is deduplicated.
- A meaningful fraction of the web is mirrored/syndicated/templated content — storing
  every byte under every URL wastes storage and downstream indexing work.
- → justifies content-hash-addressed blob storage so identical bytes are stored once
  regardless of how many URLs point to them (Deep Dive 4).

Frontier size (why it can't live in memory):

- At steady state the frontier can hold **billions** of discovered-but-not-yet-fetched
  URLs (out-degree growth outpaces fetch throughput during a fast expansion phase).
- → rules out an in-process seen-URL set or in-memory-only queue; both the dedup index
  (Deep Dive 2) and the frontier itself need a durable, horizontally-scalable store.

Fetcher fleet size (why politeness, not raw throughput, is the real ceiling):

- At ~1,200 pages/sec peak with ~300ms average fetch latency, the in-flight concurrency
  needed is ~1,200 × 0.3 ≈ 360 concurrent fetches — comfortably inside a modest fleet
  (a handful of instances holding hundreds of concurrent connections each).
- → the fleet's *raw* capacity is not the bottleneck; Deep Dive 1's per-host
  `Crawl-delay` enforcement is what actually paces the crawl, since any single
  link-heavy host is rate-limited to ~1 request/delay regardless of how much idle
  fetcher capacity exists. Adding more fetchers speeds up *breadth* across hosts, not
  how fast any one host can be crawled.

Bandwidth (why fetcher count is also a network-capacity question, not just a
concurrency one):

- 1,200 pages/sec × ~500KB/page ≈ 600MB/sec ≈ 4.8Gbps of sustained download bandwidth
  at peak.
- A conservative assumption is that only ~25% of an instance's advertised bandwidth is
  usable for downloads once other traffic (control-plane, health checks, co-located
  services) is accounted for — a network-optimized cloud instance rated at ~25Gbps
  therefore yields ~6Gbps usable, comfortably covering the 4.8Gbps requirement on a
  small number of instances once accounting for headroom and peak bursts above the
  sustained average.
- → at this design's scale, bandwidth happens not to be the forcing function the way it
  can be for a one-shot, deadline-bound crawl of the entire web in days rather than a
  continuous trickle — but it's exactly the kind of number worth checking explicitly
  before assuming "add more fetcher instances" is free, since fetcher count is bound by
  whichever of {concurrency, bandwidth, per-host politeness} is tightest, not just one
  of them.

## 3. Core Entities

- **FrontierEntry** — a discovered-but-not-yet-fetched URL, carrying its host,
  priority, and the earliest time it's allowed to be fetched (politeness).
- **Host** — a crawl target host's `robots.txt` rules, crawl-delay, and the next time
  that host may be contacted.
- **SeenURL** — the durable dedup record for a URL already enqueued or fetched at least
  once (prevents re-discovering and re-enqueuing the same URL endlessly).
- **PageRecord** — the result of a fetch: content hash, where the raw bytes live, where
  the *extracted* text ends up once parsed, content type, pipeline status, last-crawled
  time, and the recrawl priority signal.
- **ParseTask** — a queue-only message (`UrlHash`, raw storage link) handed from the
  Fetcher to a decoupled Parser once raw bytes are durably stored; never persisted past
  consumption (Deep Dive 3).

## 4. API Design

```
# Seed/operational input — not a high-QPS public API
POST /seeds
body: { urls: [...], priority }
  -> { accepted: N }

# Monitoring / debugging
GET /pages/{urlHash}
  -> PageRecord  { url, contentHash, storageLink, lastCrawledAt, status }

GET /hosts/{host}/status
  -> { queueDepth, lastFetchedAt, robotsRules, nextAllowedFetchTime }
```

This system has almost no public-facing API surface — it's a background pipeline, not
a request/response service. `POST /seeds` is the only real "write" a human or another
system issues; everything else (discovery, re-enqueuing, recrawl scheduling) happens
internally, which is why the HLD below is best understood as a trace of one URL's
journey through the pipeline rather than a request/response flow.

## 5. High-Level Design

Trace one seed URL through the basic pipeline. An operator (or another system) calls
`POST /seeds`, and the **Crawler Service** writes a `FrontierEntry` for it into the
**Crawl Metadata DB** (this single DB, in the basic design, holds the frontier, the
`Host` table, and `PageRecord`s). The frontier behaves as a **queue (breadth-first),
not a stack (depth-first)**: a page's own links are appended to the back, not pushed
to the front, so the crawl advances roughly one "layer" of the link graph at a time
instead of chasing one page's first link arbitrarily deep before touching its
siblings. This matters because pages overwhelmingly link back to other pages on the
*same* host; a BFS-shaped traversal naturally revisits a given host again soon after
the host's DNS/robots caches (Deep Dive 1, Deep Dive 6) were just warmed, where a
DFS-shaped traversal would wander off-host and let those caches go cold before coming
back. BFS also sidesteps a real implementation problem DFS would create: DFS needs a
*stack*, and a distributed stack with correct backtracking across billions of URLs —
popping back to exactly where a branch left off, consistently, across many
machines — is an awkward, largely bespoke thing to build, whereas a distributed
*queue* (BFS's natural structure) is a well-solved primitive with no backtracking
state to maintain (a partitioned Kafka topic, SQS, or this design's own two-tier
frontier). The same service dequeues ready entries, fetches the
page over HTTP, parses the HTML to extract outbound links, writes each newly-discovered
link back into the frontier (after checking it isn't already a known `SeenURL`), and
stores the raw page bytes in **Blob Storage (S3)**, recording the storage link and a
content hash in a new `PageRecord`. Fetcher and Parser instances run in the same
region as the Blob Storage bucket and Crawl Metadata shards they write to — this is
the highest-volume write path in the system, and a cross-region hop on every single
page would add needless latency and egress cost to it.

Parsing is dispatched by content type through a small registry (HTML parser, PDF
extractor, image handler, …) rather than one monolithic parser — this is what makes
"extensibility to new content types" (an NFR) a matter of registering a new handler,
not touching the fetch/frontier loop itself.

**Database Selection: relational vs. NoSQL for the Frontier + Metadata store**

The dominant access pattern is a point lookup/dequeue by `Host` (politeness ordering)
and a point lookup by `UrlHash`/`ContentHash` (dedup) — never a multi-entity join or an
ad-hoc filter across unrelated hosts. At ~4,000+ candidate URL writes/sec sustained
(BOTE above), with dedup checks on the same path and a hard requirement that the store
partition cleanly by host for politeness (Deep Dive 1).

**Bad: a single relational primary** holding frontier entries, hosts, and page
records. Fsync-bound commit throughput (thousands-to-tens-of-thousands of writes/sec at
best) is already a stretch once dedup lookups and per-host politeness bookkeeping share
the same write path, and there's no natural partitioning story — it would need manual
sharding bolted on regardless.

**Good: manually sharded relational, partitioned by host hash.** Solves the raw
throughput ceiling, but re-sharding as the crawl grows (and as host traffic is wildly
uneven — Deep Dive 5's point about hot hosts) becomes an ongoing operational burden, for
a workload that never actually needed SQL's join/transaction capabilities in the first
place.

**Best (chosen): a wide-column/NoSQL store (Cassandra or DynamoDB), partitioned by
`Host` for frontier entries and by `UrlHash`/`ContentHash` for the dedup tables.**
Auto-shards as the dataset grows, and the partition-key model maps directly onto the
actual access pattern (dequeue-by-host, lookup-by-hash) with zero joins needed anywhere
in this design.

**Trade-off:** no ad-hoc cross-host queries (e.g., "all pages crawled in the last hour
across every host" needs a separate export/analytics path, not a live scan) and only
eventual consistency on frontier writes — a URL can rarely be enqueued twice under
concurrent discovery. Compensated by making fetch-and-mark idempotent: re-fetching an
already-fetched URL wastes one fetch, it doesn't corrupt anything.

That's the entire basic design: **Client submits seeds → Crawler Service dequeues,
fetches, parses, re-enqueues new links → Crawl Metadata DB (frontier + hosts + page
records) ↔ Blob Storage for raw bytes.** It satisfies the functional requirements as
long as you ignore scale and the web's adversarial edges. It does **not** yet handle:

- never overwhelming any single host while still processing millions of hosts in
  parallel (politeness at scale),
- detecting an already-seen URL without holding billions of URLs in memory,
- a single Crawler Service instance being both a throughput ceiling and a single point
  of failure,
- storing identical or near-identical content only once,
- a maliciously or accidentally infinite URL space (crawler traps),
- DNS resolution becoming a serialization bottleneck at high fetch concurrency,
- deciding which already-crawled pages are worth re-crawling, and how often,
- a failed fetch (timeout, 5xx) being retried sanely instead of either dropped silently
  or hammered immediately, and a page that will never succeed being given up on,
- a slow/crashing *parse* step forcing a full re-fetch, instead of just a re-parse, to
  recover (combining fetch and parse in one step makes the expensive half of the work —
  the network fetch — as fragile as the cheap half).

Each of those gaps is exactly one of the deep dives below.

## 6. Deep Dives

### Deep Dive 1 — Politeness at Scale: the Two-Tier URL Frontier

**The problem.** The crawler must sustain thousands of fetches/sec in aggregate while
never issuing more than ~1 request/`Crawl-delay` against any single host. A single
shared FIFO queue can't do both: if one host happens to dominate the queue (a large,
densely-linked site just got crawled and spawned thousands of new links), its entries
crowd out every other host's turn, and nothing in a plain FIFO enforces a minimum gap
between two requests to the *same* host.

**Bad: one global FIFO queue, dequeued by however many workers are free.** Nothing
prevents two workers from picking two URLs on the same host back-to-back, and a
link-heavy host can flood the queue, starving every other host's URLs behind it.

**Good: one dedicated queue per host.** Politeness is now enforceable (one worker per
host, following its crawl-delay) — but with tens of millions of hosts, a worker
iterating round-robin over every host queue wastes most of its time checking host
queues that aren't ready to be polled yet, and there's no way to prioritize an
important host's queue over a low-value one.

**Best (chosen): Mercator-style two-tier frontier.** A small number of **front
queues**, selected by *priority* (a page's importance/recrawl score), feed into a much
larger set of **back queues**, one per currently-active host, each holding only that
host's pending URLs. A mapping table routes a dequeued URL into its host's back queue.
Workers pull from a **min-heap keyed by `(host, nextAllowedFetchTime)`**: the heap root
is always the next host that's actually allowed to be fetched *right now*, so a worker
never wastes a cycle checking a host that's still in its politeness cooldown.

**Trade-off:** the host→back-queue mapping and the readiness heap are extra state that
must itself scale and be kept consistent as hosts are added/retired from the active set
— more bookkeeping than a single queue, but it's the only structure that gets both
properties (global throughput and per-host politeness) at once.

**Operational note — jitter.** When many URLs for the same host land back-to-back in
the frontier (common right after crawling a link-heavy page), naively re-checking them
on a fixed interval can synchronize multiple workers into hammering the same
`nextAllowedFetchTime` boundary at once. Adding a small random jitter to each
re-enqueue delay spreads that out, avoiding a self-inflicted thundering herd against a
host the design is specifically trying to be polite to.

**Operational note — default crawl-delay, and identifying ourselves.** Most hosts'
`robots.txt` don't specify an explicit `Crawl-delay` at all — `Host.CrawlDelay` defaults
to a conservative floor (e.g., 1 request/sec) rather than treating "unspecified" as "no
limit." Separately, the Fetcher sends a distinctive `User-Agent` identifying this
crawler (the same way `Googlebot` or `bingbot` identify themselves) — this is what lets
a site owner's `robots.txt` target rules at this crawler specifically, and is itself
part of being a good citizen of the sites being crawled, not just an implementation
detail.

### Deep Dive 2 — Duplicate URL Detection at Web Scale

**The problem.** Before enqueuing a newly-discovered link, the crawler must check "have
I already seen this URL" — otherwise the same popular page gets re-discovered and
re-enqueued from every page that links to it, multiplying frontier writes many times
over. At billions of URLs, an exact in-memory hash set doesn't fit on one machine, and
a per-check database round-trip against a huge table is expensive if it happens on
every single discovered link.

**Prerequisite — normalize before hashing.** `UrlHash` is only a useful dedup key if
syntactically-different URLs that point to the *same* page hash the same way. Before
computing it, the Parser normalizes each extracted link: lowercase the scheme and host,
add a missing `http://` scheme, strip a default port (`:80`/`:443`), and collapse a
trailing-slash-only difference (`/page` vs. `/page/`). Skipping this step would let the
Bloom filter and `SeenURL` table both report "not seen" for a URL that's actually a
duplicate in every way that matters, defeating Deep Dive 2 entirely regardless of how
well-tuned the dedup structures themselves are. The Parser also drops links whose
extension clearly indicates non-target content (`.jpg`, `.mp3`, `.mp4`, …) before they
ever reach the frontier — cheaper than discovering the same thing from a `Content-Type`
check after a wasted fetch.

**Bad: a durable key-value lookup (`SeenURL` table) on every single discovered link,
no filtering in front of it.** Correct, but puts the full discovery write-and-lookup
rate directly on the dedup store's read path with no cheap short-circuit — at 4,000+
discoveries/sec (BOTE) this is a lot of avoidable point lookups for links that usually
*aren't* duplicates in a growing crawl.

**Good: an in-memory Bloom filter as a fast pre-check.** A Bloom filter answers
"definitely not seen" (skip the DB lookup, enqueue immediately) or "maybe seen" (fall
through to a definitive check) using a tiny memory footprint regardless of URL count,
with a tunable false-positive rate. Fast, but a filter living only in one process's
memory is lost on restart (every URL looks "maybe new" again, hammering the durable
store right after a redeploy) and doesn't horizontally scale with the frontier itself.

**Best (chosen): Good, backed by a durable, sharded `SeenURL` table keyed by
`UrlHash`.** The Bloom filter short-circuits the common case (not seen → straight to
enqueue); a "maybe seen" hit falls through to the definitive `SeenURL` lookup, which
also persists across restarts. The filter is rebuilt/warmed from the durable table on
startup (or periodically snapshotted) instead of starting cold.

**Trade-off:** a Bloom filter can produce **false positives** (claims "maybe seen" for
a URL that's actually new, costing one extra durable lookup — never a correctness bug),
but **never false negatives** (it will never wrongly claim "definitely not seen" for a
URL that was seen) — so the only cost of tuning the filter smaller is more fallback
lookups, never missed dedup.

### Deep Dive 3 — Fetch Reliability: Decoupling, Retries, and Horizontal Scaling

**The problem.** A single Crawler Service instance that fetches, parses, and writes
frontier updates in one step for every host has three separate failures bundled into
one: (a) it's a throughput ceiling (one machine's network/CPU can't sustain thousands
of fetches/sec against millions of hosts), (b) it's a single point of failure, and (c)
because fetch and parse aren't separated, a crash or bug in the *cheap* half of the work
(parsing) forces redoing the *expensive* half (the network fetch) to recover — and
there's no defined behavior yet for a fetch that fails outright (timeout, connection
reset, 5xx).

**3a. Decoupling fetch from parse.**

**Bad: fetch and parse in one step, bytes handed directly from fetch to parse in
memory, nothing written durably until parsing finishes.** If the parse step crashes,
hangs, or has a bug partway through, the already-completed network fetch is wasted and
must be redone — for a workload where the network round-trip, not the parse, is the
expensive and unreliable part. It also means reprocessing every page with an updated
parser (e.g., the ML team now also wants `alt` text from images) requires a full
re-crawl instead of just re-running the parse step.

**Best (chosen, single addition — no real alternative worth ranking): split into two
decoupled stages connected by a durable `ParseTask` queue.** The **Fetcher** does
exactly one job — fetch the page and immediately write the raw bytes to Blob Storage —
then enqueues a tiny `ParseTask` (just the `UrlHash` and the raw storage link) onto a
separate queue. A decoupled **Parser** fleet consumes that queue, reads the already-
durable raw bytes, extracts text/links, writes extracted content back to Blob Storage,
and updates `PageRecord`. A parse crash now only costs replaying the small `ParseTask`
message, never a re-fetch; each stage also scales and gets deployed independently
(Parser logic changes don't touch the Fetcher at all), and reprocessing every page with
new extraction logic is a queue replay, not a re-crawl.

**Trade-off:** one more durable hop (the `ParseTask` queue) and brief end-to-end lag
between "fetched" and "parsed" status on a `PageRecord` — acceptable given the NFR that
bounded staleness is fine, and a clear net win for both fault isolation and the ability
to evolve parsing logic without re-touching the network.

**A second, orthogonal extensibility axis.** The HLD's content-type plugin registry
(HTML parser, PDF extractor, …) extends the design along *content format*. The
`ParseTask` queue extends it along a different axis for free: multiple independent
consumer groups can subscribe to the same raw-fetch events — a duplicate-content
detector, a malware/phishing scanner, a copyright-violation checker — each reading the
same durably-stored raw bytes without touching the Parser or the Fetcher at all. Adding
one of these later is "stand up a new consumer group," not "modify the core pipeline."

**3b. Retries and dead-letter handling.**

**Bad: no retry at all, or an in-memory fixed-delay retry with no cap.** Dropping a
failed fetch silently under-crawls; an uncapped immediate retry against a host that's
down or rate-limiting just adds more load to an already-struggling server, violating
politeness.

**Good: exponential backoff on fetch failure, re-enqueued into the Fetcher's input with
an increasing delay each attempt** (e.g., 30s → 2m → 5m → 15m). Bounds retry pressure
on a struggling host, but still retries forever for a URL that will never succeed (a
permanently dead domain, a typo'd URL) — wasted, unbounded retry budget.

**Best (chosen): Good, plus a hard retry cap (e.g., 5 attempts) that routes the URL to
a dead-letter queue instead of retrying indefinitely.** A URL that exhausts its retries
is marked `PageRecord.Status = 'failed'` and stops consuming fetch budget; the
dead-letter queue is inspectable separately (most entries are simply dead
domains/broken links, not a design flaw to fix).

**Trade-off:** a page genuinely worth having that fails 5 times in a row due to
transient bad luck (not a permanently dead site) is given up on rather than retried
forever — an acceptable bound given that unbounded per-URL retry budget would let a
long tail of dead links quietly consume fetch capacity that live pages need.

**3c. Horizontal scaling.**

**Good: a stateless fleet of fetcher workers (and, per 3a, a separately-scaled parser
fleet) behind the frontier, scaled by adding instances.** Removes the single-instance
throughput ceiling — but if every worker can pull from any host's back queue, two
workers can still race on the same host unless the readiness heap (Deep Dive 1) itself
is the single shared coordination point, which then has to be made safe for concurrent
access.

**Best (chosen): Good, plus sharding host ownership by `hash(host) mod N` across
fetcher-worker groups.** Each worker group owns a disjoint slice of hosts (and their
back queues/readiness heaps), so there's no cross-worker contention over which host to
fetch next — ownership, not locking, prevents two workers from racing the same host.
Adding fetch capacity is just adding workers and reshuffling host ownership, the same
shape as adding shards to the Frontier DB (Deep Dive 2's store, Deep Dive 1's queues).

**Trade-off:** a worker crash temporarily strands its shard of hosts until ownership is
reassigned (a short availability gap for those hosts specifically, not the whole
crawl) — acceptable given the NFR that availability matters more than any single page's
crawl timing, as long as reassignment is bounded and automatic.

**Operational note:** the Fetcher issues a lightweight `HEAD` request first and checks
the response's `Content-Length` header before ever issuing the full `GET` — skipping
or deferring pages past a configured size ceiling without paying for the download at
all. A handful of pathologically large pages (video files served without a correct
content type, massive generated dumps) would otherwise tie up a fetch worker and
bandwidth disproportionate to their value as crawl targets.

**Cross-cutting: why all of this is safe to retry.** Deep Dive 1's occasional
double-enqueue under eventual consistency, 3a's replayed `ParseTask` after a parser
crash, and 3b's retried fetch after a timeout all rely on the same underlying
guarantee: **every write in this pipeline is idempotent.** A `PageRecord` write is
keyed by `UrlHash` and a blob write is keyed by `ContentHash` — processing the same
URL or the same bytes twice overwrites the same key with the same value rather than
creating a duplicate or corrupting state. This is what makes "at-least-once" an
acceptable processing guarantee everywhere in this design instead of needing
exactly-once semantics (distributed transactions, dedup tokens on every hop) — the
occasional wasted re-fetch or re-parse costs time, never correctness.

### Deep Dive 4 — Duplicate and Near-Duplicate Content Detection

**The problem.** Different URLs frequently serve byte-identical or near-identical
content (mirrors, syndicated articles, templated pages that differ only in
boilerplate/navigation). Storing and later indexing every copy separately wastes
storage (BOTE: 500TB/month uncompressed) and pollutes downstream search results with
redundant pages.

**Bad: store every fetched page under its own URL, no dedup at all.** Simple, but pays
full storage and indexing cost for content that already exists elsewhere under a
different URL.

**Good: exact-match dedup via a content hash (e.g., SHA-256) of the raw bytes.**
Catches byte-identical mirrors cheaply — if the hash already exists in Blob Storage, the
new `PageRecord` just points at the existing object instead of writing a new one. Misses
near-duplicates: two pages that differ only in a timestamp or an ad slot hash to
completely different values despite being "the same" content for indexing purposes.

**Best (chosen): Good, plus SimHash (or MinHash) fingerprinting with
locality-sensitive hashing (LSH) for near-duplicate detection.** SimHash produces
similar fingerprints for similar documents (small edit distance → small Hamming
distance between fingerprints); LSH buckets similar fingerprints together so finding
"is anything already close to this one" is a cheap bucket lookup, not an all-pairs
comparison. Near-duplicates get flagged and can be stored once with multiple URL
references, same as exact duplicates.

**Trade-off:** near-dup detection adds real compute per page (fingerprinting + an LSH
lookup) and a tunable similarity threshold that can occasionally merge two genuinely
distinct-but-similar pages (or miss a near-dup that's just past the threshold) — a
precision/recall knob, not a free win, tuned conservatively to avoid incorrectly
collapsing distinct content.

### Deep Dive 5 — Crawler Traps and Unbounded URL Spaces

**The problem.** The web contains both adversarial and accidental infinite URL spaces —
a calendar widget generating an endless chain of "next day" links, a session-ID or
query-param combinatorial explosion, a deliberately constructed trap meant to waste
crawler resources. Left unchecked, one such host can consume disproportionate fetch
budget indefinitely and never converge.

**Bad: no limits at all.** The crawler happily follows every discovered link forever;
one trap host can monopolize a fetcher shard's time indefinitely.

**Good: hard caps — max crawl depth from a seed, and a max pages-per-host-per-day
budget.** Bounds the damage any single host can do, but a fixed depth/budget is
somewhat arbitrary: it can cut off a legitimately large, deep site at the same point it
cuts off an actual trap.

**Best (chosen): Good, plus pattern-based trap detection on the URL path/query
structure** — e.g., recognizing a repeating path template with only a monotonically
incrementing or date-like parameter changing, and routing further matches from that
pattern into a heavily-deprioritized or capped sub-queue instead of the normal
priority flow. This targets the actual signature of a trap (structural repetition) in
addition to blunt caps.

**Trade-off:** pattern-based detection is a heuristic and can false-positive on a
legitimately large paginated site (e.g., a real archive with thousands of genuinely
distinct dated pages) — mitigated by deprioritizing rather than hard-blocking a
detected pattern, so a real site still gets crawled, just more slowly than an
undetected one would.

### Deep Dive 6 — DNS Resolution as a Hidden Bottleneck

**The problem.** A naive fetch path resolves DNS synchronously per request. DNS
round-trip latency (tens to hundreds of ms, worse for a cold/uncached name) directly
serializes with fetch latency; at thousands of concurrent fetches, blocking a worker
thread on a synchronous DNS call wastes exactly the concurrency the fetcher fleet
exists to provide.

**Single addition, no real alternative worth ranking: asynchronous DNS resolution with
local caching, respecting each record's TTL.** Fetcher workers issue non-blocking DNS
lookups and move on to other work while a resolution is in flight; resolved addresses
are cached locally (and in a shared cache like Redis across a worker shard) so repeat
fetches to the same host skip resolution entirely until the TTL expires.

**Trade-off:** caching DNS answers for their TTL window means a host that changes IP
mid-TTL can get a handful of failed/misrouted fetch attempts until the cache entry
naturally expires and re-resolves — a small, bounded, self-correcting staleness window,
not a correctness bug the design needs to otherwise guard against.

**Operational note — provider redundancy.** At fleet scale, a single third-party DNS
provider's own rate limits can themselves become a bottleneck or a single point of
failure. Resolving against multiple providers round-robin (not just one) spreads load
across them and means one provider's outage or throttling doesn't stall resolution
fleet-wide — a cheap, practical addition on top of caching, not a replacement for it.

### Deep Dive 7 — Freshness and Recrawl Prioritization

**The problem.** Once a page has been crawled once, "crawl it again eventually" needs
an actual policy — recrawling every page on a fixed global interval either wastes
budget on pages that rarely change or lets fast-changing pages (news, pricing) go
stale for too long between visits.

**Good: a fixed recrawl interval per page (e.g., every 7 days).** Simple and bounded,
but blind to each page's actual behavior — a news homepage that changes hourly and a
static "About Us" page get the same treatment.

**Best (chosen): a priority score combining page importance (e.g., inbound link
count) with an observed change-frequency estimate, updated each time the page is
re-fetched** (did the content hash change since last crawl, and how long had it been
since the previous crawl). Higher-scoring pages get re-inserted into the frontier's
front queues (Deep Dive 1) sooner; rarely-changing, low-importance pages drift toward
the back of the priority order naturally, without needing a separate scheduling system.

**What "changed" means operationally:** a recrawl compares the new fetch's content hash
(Deep Dive 4) against the `PageRecord`'s last stored hash — identical hash moves the
change-frequency estimate toward "slower," a different hash moves it toward "faster,"
using a simple exponential moving average so one anomalous fetch doesn't swing the
estimate wildly.

**Cheaper recrawl check — `HEAD` before `GET`.** On a *recrawl* specifically (not a
first fetch), the Fetcher issues a `HEAD` request first and compares the `Last-Modified`
header against the `PageRecord`'s stored value from the previous crawl. An unchanged
`Last-Modified` skips the full `GET` and content-hash comparison entirely — a real
bandwidth/compute saving for the large share of recrawled pages that haven't actually
changed. This is a pre-check only, not a replacement for Deep Dive 4's content hash:
many servers omit `Last-Modified` or set it unreliably (e.g., always to request time),
so its absence or a changed value just falls through to the normal full fetch and hash
comparison, never the reverse — a page is never skipped on `Last-Modified` alone
without a `GET` ever having established its initial content hash.

**Trade-off:** a freshly-discovered page has no change-frequency history yet, so its
initial recrawl interval is a default guess (e.g., treat as "unknown, moderate
priority") until enough observed recrawls accumulate real signal — meaning early
recrawls of brand-new pages are somewhat wasted probes rather than fully-informed
decisions, unavoidable for any page the system hasn't observed changing yet.

## 7. Data Model

```
FrontierEntry                          (per-host back queue item, Deep Dive 1)
- Host                  (partition key)
- Url
- Priority              -> from Deep Dive 7's freshness/importance score
- DiscoveredAt

Host                                    (politeness state, Deep Dive 1)
- Host                  (partition key)
- RobotsRules
- CrawlDelay
- NextAllowedFetchTime  -> read by the readiness heap

SeenURL                                 (dedup, Deep Dive 2)
- UrlHash               (partition key)
- FirstSeenAt

PageRecord                              (fetch result + recrawl signal)
- UrlHash               (partition key)
- ContentHash           -> Deep Dive 4 dedup key into Blob Storage
- RawStorageLink        -> written by Fetcher, before parsing (DD3a); bytes stored
                           compressed, decompressed only on read
- ExtractedTextLink     -> written by Parser, after parsing (DD3a)
- ContentType           -> dispatches the extraction plugin (HLD)
- Status: 'pending' | 'fetched' | 'parsed' | 'failed'   -> DD3b retry/DLQ state
- RetryCount            -> DD3b exponential backoff + DLQ threshold
- LastCrawledAt
- LastModifiedHeader    -> DD7's cheap HEAD pre-check before a full recrawl fetch
- ChangeFreqEstimate    -> Deep Dive 7's EMA
- PriorityScore         -> feeds back into FrontierEntry.Priority on recrawl

ParseTask                               (queue-only message, DD3a)
- UrlHash
- RawStorageLink        -> not stored durably past consumption by the Parser

ContentHash -> CanonicalUrlHash          (dedup index, Deep Dive 4)
- ContentHash           (partition key)
- CanonicalUrlHash      -> the first URL that stored this content; later matches
                           reference it instead of storing again
```

`Host` is the partition key for `FrontierEntry` specifically because politeness (Deep
Dive 1) and fetcher sharding (Deep Dive 3) both need all of one host's pending work
colocated — the same key decision that makes ownership-based sharding possible without
cross-shard coordination for any single host's queue.

## 8. What We Deliberately Did NOT Add

- **A separate message queue (Kafka) in front of the frontier.** The frontier itself
  (Deep Dive 1's two-tier queue structure) already provides durable, ordered,
  host-partitioned buffering — layering a generic pub/sub queue in front of it would
  duplicate that job without adding a capability the design needs.
- **A CDN.** The crawler is a consumer of other hosts' content, not a server of its
  own static assets to end users — there's no edge-caching role for a CDN to play here.
- **Headless browser rendering in the core pipeline.** JS-rendered content is treated
  as a separate, optional, heavier downstream stage (Out of Scope) rather than baked
  into every fetch, since most of the web is still served or meaningfully crawlable
  without full rendering.
- **A generic distributed lock service for frontier coordination.** Deep Dive 3's
  ownership-by-hash sharding avoids needing cross-worker locks for "who fetches this
  host next" in the first place.
- **Per-domain outbound proxy rotation to parallelize fetches against a single host.**
  Doesn't actually solve anything here: a single large site can yield millions of URLs
  on its own domain, and politeness (Deep Dive 1) caps how fast that *one* host can be
  fetched regardless of how many different source IPs the requests come from — adding
  proxies would just be more infrastructure pointed at a ceiling that isn't
  IP-related.

## 9. Potential Follow-Up Deep Dives

- **Redistributing host ownership smoothly** when fetcher-worker shard count changes
  (Deep Dive 3) — a naive `hash(host) mod N` remap reshuffles nearly every host's
  ownership when `N` changes; consistent hashing would bound the remap to a small
  fraction of hosts, the same problem a sharded cache or database faces when resizing.
- **Detecting and handling `robots.txt` or redirect-chain abuse** more deeply —
  e.g., a host whose `robots.txt` itself times out or redirects indefinitely, which
  sits adjacent to but isn't the same failure mode as a content-level crawler trap
  (Deep Dive 5).
- **Cross-region crawling** — running fetcher fleets in multiple regions (closer to
  geographically distributed hosts, for lower per-fetch latency) and how that interacts
  with a single logical frontier's politeness state for a host that could be fetched
  from more than one region. Two sub-questions worth resolving deliberately: (1) a
  domain's TLD or registrar location does **not** reliably indicate where it's actually
  hosted, so naively routing fetchers by domain-guessed geography mostly doesn't pay
  off — fetcher placement should follow observed latency, not domain name heuristics;
  (2) the dedup state itself (Deep Dive 2's `SeenURL`, Deep Dive 4's `ContentHash`
  index) faces the same centralized-vs-decentralized choice a multi-region cache
  always does — a single strongly-consistent store is simple but adds cross-region
  latency to every dedup check from a remote region, while a CRDT-style eventually-
  consistent set per region avoids that hop at the cost of occasionally processing the
  same URL/content twice before replicas converge — tolerable here specifically
  because every write is already idempotent (Deep Dive 3's cross-cutting note).
- **A concrete threshold for Deep Dive 5's trap deprioritization** — what specific
  signal (e.g., N consecutive matches of the same path pattern within a time window)
  actually triggers routing a host's traffic into the deprioritized sub-queue, versus
  leaving it in normal priority flow.
- **Cross-stage observability.** Per-stage dashboards and alerting — frontier depth,
  fetch success rate, parse queue lag, Dead-Letter Queue growth (paging on-call rather
  than requiring a manual check), and fetcher-fleet CPU/RAM/bandwidth saturation —
  would make it possible to spot a stage falling behind, or a job-scheduler/backoff
  failure, before it becomes an availability problem. Not modeled in detail here, but a
  real production gap for a multi-stage pipeline like this one.

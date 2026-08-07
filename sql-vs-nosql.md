# SQL vs NoSQL — System Design Deep Dive

> **Goal:** Understand not just *what* the differences are, but *why* they exist and *when* each is the right call in an interview.

---

## Table of Contents

1. [The Mental Model](#the-mental-model)
2. [How SQL Works Internally](#how-sql-works-internally)
3. [How NoSQL Works Internally](#how-nosql-works-internally)
4. [Scaling: The Real Difference](#scaling-the-real-difference)
5. [CAP Theorem & Consistency Models](#cap-theorem--consistency-models)
6. [Data Modeling Differences](#data-modeling-differences)
7. [NoSQL Database Types](#nosql-database-types)
8. [Decision Framework](#decision-framework)
9. [Common Interview Scenarios](#common-interview-scenarios)
10. [Convincing Interview Phrases](#convincing-interview-phrases)
11. [Numbers to Know](#numbers-to-know)
12. [Summary Cheat Sheet](#summary-cheat-sheet)

---

## The Mental Model

The core difference is not performance. It is **what guarantees each database makes and what it gives up in exchange.**

```
SQL                                     NoSQL
─────────────────────────────────       ─────────────────────────────────
Rich data model (JOINs, FK)             Simple data model (no JOINs)
Strong consistency (ACID)               Tunable consistency (eventual → strong)
Hard to scale writes horizontally       Designed for horizontal write scale
Schema enforced                         Schema flexible
Query anything (SQL)                    Query by known access patterns only
```

**One-line summary:**
> SQL trades horizontal scalability for a rich, consistent data model.
> NoSQL trades the rich data model for horizontal scalability.

Neither is universally better. You are always choosing which trade-off fits your problem.

---

## How SQL Works Internally

Understanding why SQL is hard to scale requires understanding how it works.

### Storage: Row-Oriented, Heap + Indexes

```
Table: posts
┌────┬─────────┬──────────────────┬─────────────────────┐
│ id │ user_id │ content          │ created_at          │
├────┼─────────┼──────────────────┼─────────────────────┤
│  1 │    42   │ Hello world      │ 2024-01-01 10:00:00 │
│  2 │    17   │ Another post     │ 2024-01-01 10:01:00 │
│  3 │    42   │ My second post   │ 2024-01-01 10:05:00 │
└────┴─────────┴──────────────────┴─────────────────────┘

B-tree Index on user_id:
user_id=17 → [row 2]
user_id=42 → [row 1, row 3]
```

- Data lives in a heap file (unordered pages on disk)
- Indexes are separate B-tree structures that point back to heap rows
- Every write must update the heap AND every index on that table

### ACID via WAL + MVCC

```
Write flow:
  BEGIN
    ↓
  Write to WAL (Write-Ahead Log) — durable immediately
    ↓
  Update in-memory buffer
    ↓
  COMMIT → WAL flushed to disk → transaction acknowledged
    ↓
  Buffer eventually flushed to heap (background)
```

- **WAL** ensures durability — if the server crashes, the WAL replays to recover
- **MVCC** (Multi-Version Concurrency Control) keeps old row versions so readers don't block writers
- Both add write overhead that cannot be avoided

### JOINs Require Data Colocation

```sql
SELECT u.username, p.content
FROM users u
JOIN posts p ON p.user_id = u.id
WHERE u.id = 42;
```

This works because both `users` and `posts` live on the **same machine**. The database can efficiently look up both tables, follow the index, and join in memory.

Once you shard across machines, `users` might be on Node A and `posts` on Node B — the JOIN now requires a network call, which is orders of magnitude slower.

---

## How NoSQL Works Internally

NoSQL databases are built around the assumption that data will be **sharded across many machines from day one.**

### The Fundamental Design Choice: Denormalization

Instead of splitting data into normalized tables and joining at query time, NoSQL **stores everything together** that will be queried together.

```
SQL (normalized):                    NoSQL (denormalized):
                                     
users table:                         Single document in MongoDB:
{ id: 42, username: "alice" }        {
                                       _id: "post:1",
posts table:                           user_id: 42,
{ id: 1, user_id: 42, content: ... }   username: "alice",   ← duplicated!
                                       content: "Hello",
comments table:                        comments: [          ← embedded!
{ id: 1, post_id: 1, text: ... }         { user: "bob", text: "Hi" }
                                       ]
                                     }
```

**Trade-off:**
- No JOIN needed — one read gets everything
- But if Alice changes her username, you must update it in every document that stored it

### Partitioning (Sharding) Is Native

```
Cassandra ring — 4 nodes, token ranges:

         Node A (0–25%)
        /              \
Node D (75–100%)    Node B (25–50%)
        \              /
         Node C (50–75%)

Write: row with partition key "user:42"
  → hash("user:42") = 31% → goes to Node B
  → automatically replicated to Node C (next node) for fault tolerance
```

Adding a 5th node:
- The ring rebalances automatically
- Each existing node gives up a portion of its token range
- No application changes needed

This is fundamentally different from SQL sharding, where you must:
1. Manually pick a shard key
2. Manually route queries in application code
3. Manually handle rebalancing when adding nodes

---

## Scaling: The Real Difference

This is the most important section for interviews.

### SQL Scaling Path

```
Step 1: Vertical scaling
        → Bigger machine (more RAM, faster CPU, NVMe SSD)
        → Works well up to a point (~$50K/month server)

Step 2: Read replicas
        → Add replica nodes for read traffic
        → Primary handles writes, replicas handle reads
        → Easy to add, but writes are still single-node

Step 3: Connection pooling (PgBouncer)
        → PostgreSQL is expensive per-connection (~5MB RAM each)
        → Pooler multiplexes thousands of app connections → fewer DB connections

Step 4: Caching (Redis)
        → Put hot data in Redis, skip the DB entirely for reads
        → Dramatically reduces DB read load

Step 5: Table partitioning
        → Split a huge table into smaller physical pieces by range or hash
        → Each partition is smaller → indexes are smaller → queries faster

Step 6: Manual sharding ← THIS IS WHERE IT GETS PAINFUL
        → Split data across multiple PostgreSQL instances
        → Application must route to the right shard
        → JOINs across shards = impossible or very slow
        → Foreign keys across shards = impossible
        → Distributed transactions = complex (2-phase commit)
        → Rebalancing shards = operational nightmare
```

**The wall:** SQL scales reads easily. SQL scaling writes past a single node requires giving up the relational model — at which point, you might as well have used NoSQL from the start.

### NoSQL Scaling Path

```
Step 1: Start with a cluster (e.g., 3-node Cassandra)
        → Writes and reads distributed from day one

Step 2: Add nodes
        → Data automatically rebalances
        → Linear throughput increase
        → No application changes

Step 3: Add more nodes
        → Repeat indefinitely
```

**Why it works:** NoSQL gave up JOINs and cross-partition transactions. Without those constraints, sharding is trivial — every row is self-contained on its partition.

### The Write Throughput Comparison

```
Single PostgreSQL node:       ~5K–10K writes/sec
PostgreSQL + manual sharding: ~50K–100K writes/sec (painful to operate)
Cassandra (10 nodes):         ~500K–1M writes/sec (easy to operate)
DynamoDB:                     essentially unlimited (AWS manages it)
```

**Important nuance for interviews:**
> NoSQL doesn't scale *better* per node. It scales *more easily* across nodes because it operates with fewer guarantees. You are not getting something for free — you are trading consistency, query flexibility, and relational integrity for operational simplicity at scale.

---

## CAP Theorem & Consistency Models

Every distributed database must choose what to do when a network partition occurs.

### CAP Theorem

```
         Consistency
        (every read gets
         the latest write)
              /\
             /  \
            /    \
           /  CA  \
          /  (only \
         /  single  \
        / node, no   \
       /  partition)  \
      ──────────────────
     /                  \
    /                    \
CP                        AP
(consistent +             (available +
 partition-tolerant)       partition-tolerant)
Sacrifice availability    Sacrifice consistency
during partition          during partition

Examples:                 Examples:
PostgreSQL (single node)  Cassandra
HBase                     DynamoDB
Zookeeper                 CouchDB
```

**The interviewer insight:** In a real distributed system, network partitions happen. So the real choice is between **CP** (stay consistent, go down if partitioned) and **AP** (stay up, risk stale reads).

### Consistency Models in Practice

```
Strongest                                           Weakest
    │                                                   │
    ▼                                                   ▼
Linearizable    Serializable    Read-your-writes    Eventual
(real-time      (transactions   (you see your       (replicas
 ordering)       appear serial)  own writes)         converge
                                                     eventually)

PostgreSQL:     ─────────────────────────────►
                Linearizable to Serializable depending on isolation level

Cassandra:      ──────────────────────────────────────────────────►
                Tunable: ONE (eventual) to ALL (strong) consistency

DynamoDB:       ───────────────────────────────────────────────────►
                Eventually consistent reads (default) or strongly consistent
```

### Cassandra Tunable Consistency

```
Write to N replicas. Read from R replicas. Quorum when R + W > N.

N=3 replicas:
  W=1, R=1  → fastest, eventual consistency (risk of stale reads)
  W=2, R=2  → quorum, strong consistency (most common)
  W=3, R=3  → strongest, slowest (all replicas must agree)
```

**Interview answer when asked about consistency in NoSQL:**
> *"Cassandra's consistency is tunable. For user profile reads, I'd use eventual consistency for speed. For anything financial, I'd use quorum reads and writes to guarantee no stale data."*

---

## Data Modeling Differences

### SQL: Normalize, Then Join

Design to eliminate duplication. Retrieve by joining at query time.

```sql
-- 3 normalized tables
CREATE TABLE users    (id, username, email);
CREATE TABLE posts    (id, user_id, content, created_at);
CREATE TABLE comments (id, post_id, user_id, text, created_at);

-- Query joins them at read time
SELECT u.username, p.content, c.text
FROM users u
JOIN posts p ON p.user_id = u.id
JOIN comments c ON c.post_id = p.id
WHERE p.id = 123;
```

**Strength:** Data is stored once. Updates are simple. Queries are flexible.
**Weakness:** JOINs are expensive at scale. Can't shard across machines without breaking JOINs.

### NoSQL: Model Around Access Patterns

Design for how data will be read, not for normalization.

```
Step 1: Identify your access patterns first
  - Get a post with its comments → embed comments in post document
  - Get all posts by a user → partition posts table by user_id
  - Get a user's feed → pre-compute and store as a list

Step 2: Denormalize to serve those patterns without joins
```

**Cassandra example — model for "get all posts by user":**

```sql
-- Partition key = user_id → all posts for a user land on the same node
-- Clustering key = created_at DESC → rows stored in sorted order on disk

CREATE TABLE posts_by_user (
    user_id   UUID,
    created_at TIMESTAMP,
    post_id   UUID,
    content   TEXT,
    username  TEXT,     -- denormalized from users table
    PRIMARY KEY (user_id, created_at)
) WITH CLUSTERING ORDER BY (created_at DESC);

-- This query is a single partition scan — O(1) regardless of total data size
SELECT * FROM posts_by_user
WHERE user_id = ? LIMIT 20;
```

**MongoDB example — embed related data:**

```json
{
  "_id": "post:abc123",
  "user_id": 42,
  "username": "alice",
  "content": "Hello world",
  "created_at": "2024-01-01T10:00:00Z",
  "comments": [
    { "user": "bob", "text": "Great post!", "at": "2024-01-01T10:05:00Z" },
    { "user": "carol", "text": "Agreed!", "at": "2024-01-01T10:10:00Z" }
  ],
  "like_count": 42
}
```

One read returns everything. No JOIN needed.

### The Critical Difference

```
SQL:   Model the data → derive queries later
NoSQL: Know your queries → model data around them
```

If you don't know your access patterns upfront, SQL is safer — you can query it in any way after the fact. NoSQL forces you to commit to access patterns at design time.

---

## NoSQL Database Types

Not all NoSQL databases are the same. Each is optimized for a different access pattern.

### Key-Value Stores

```
Structure: key → value (opaque blob)
Examples:  Redis, DynamoDB (simple use), Amazon ElastiCache

Best for:
  - Sessions: session_id → { user_id, expiry, permissions }
  - Rate limiting: "user:42:requests" → count
  - Caching: any hot read result
  - Leaderboards: sorted sets in Redis

Not for:
  - Complex queries or filtering by value fields
  - Relationships between keys

Redis example:
  SET session:abc123 '{"user_id": 42, "role": "admin"}' EX 3600
  GET session:abc123
  INCR rate_limit:user:42:2024010110  ← atomic counter for rate limiting
```

### Document Stores

```
Structure: key → JSON document (nested, flexible schema)
Examples:  MongoDB, CouchDB, Firestore

Best for:
  - Product catalogs (different attributes per product type)
  - User profiles (varying fields per user)
  - Content management (articles, blog posts with metadata)
  - Any entity that is naturally self-contained

Not for:
  - Heavy relationships between documents
  - ACID transactions across documents (MongoDB 4.0+ has limited multi-doc transactions)

MongoDB example:
  {
    "_id": "product:laptop:42",
    "name": "MacBook Pro",
    "price": 2499.99,
    "specs": {
      "ram": "16GB",
      "storage": "512GB SSD",
      "display": "14-inch Retina"
    },
    "tags": ["apple", "laptop", "professional"]
  }
```

### Wide-Column Stores

```
Structure: rows partitioned by partition key, columns sorted by clustering key
Examples:  Cassandra, HBase, Google Bigtable

Best for:
  - Time-series data (metrics, events, logs) — partition by entity, cluster by time
  - Activity feeds — partition by user_id, cluster by timestamp
  - IoT sensor data — massive write throughput, time-ordered reads
  - Any high-write, known-access-pattern workload

Not for:
  - Ad-hoc queries (no indexes except on partition/clustering keys)
  - Relationships between partitions

Cassandra example:
  Partition key:  sensor_id   ← determines which node stores this data
  Clustering key: recorded_at ← determines row order within partition

  SELECT * FROM sensor_readings
  WHERE sensor_id = 'sensor:xyz'
    AND recorded_at > '2024-01-01'
  LIMIT 1000;
  -- Single partition scan → extremely fast
```

### Graph Databases

```
Structure: nodes and edges with properties
Examples:  Neo4j, Amazon Neptune

Best for:
  - Social graphs (friends of friends)
  - Recommendation engines (people who bought X also bought Y)
  - Fraud detection (connected accounts)
  - Knowledge graphs

Not for:
  - Simple CRUD operations
  - High write throughput

Neo4j example:
  MATCH (u:User {id: 42})-[:FOLLOWS]->(friend)-[:POSTED]->(post)
  WHERE post.created_at > datetime() - duration('P7D')
  RETURN post
  ORDER BY post.created_at DESC
  LIMIT 20;
  -- "Show me posts from people I follow in the last 7 days"
  -- A single Cypher query; in SQL this is a 3-table JOIN with subquery
```

### Search Engines (Special Category)

```
Structure: inverted index on document fields
Examples:  Elasticsearch, Apache Solr

Best for:
  - Full-text search with relevance ranking
  - Faceted search (filter by category + price range + rating)
  - Log aggregation and analysis (ELK stack)
  - Auto-complete and fuzzy matching

Not for:
  - Primary data store (treat as a read-optimized replica)
  - ACID transactions

Elasticsearch example:
  {
    "query": {
      "bool": {
        "must": { "match": { "content": "distributed systems" }},
        "filter": [
          { "term": { "category": "technology" }},
          { "range": { "published_at": { "gte": "2024-01-01" }}}
        ]
      }
    },
    "sort": [{ "_score": "desc" }]
  }
```

---

## Decision Framework

### Primary Decision Tree

```
1. Do you have complex relationships that require JOINs?
   YES → SQL (PostgreSQL)
   NO  → continue

2. Do you need ACID transactions across multiple entities?
   YES → SQL (PostgreSQL)
   NO  → continue

3. What is your primary access pattern?
   
   Key-value lookups (always by single ID)?
   └─► Key-Value Store (Redis, DynamoDB)
   
   Rich document queries, flexible schema?
   └─► Document Store (MongoDB)
   
   High write throughput, time-series, known partition key?
   └─► Wide-Column Store (Cassandra)
   
   Relationship traversal (social graph, recommendations)?
   └─► Graph DB (Neo4j)
   
   Full-text search with relevance ranking?
   └─► Search Engine (Elasticsearch)
   
   None of the above, or access patterns unknown?
   └─► SQL (PostgreSQL) — safer default, more flexible

4. What is your scale requirement for writes?
   < 10K writes/sec → SQL handles this comfortably
   10K–100K/sec    → SQL with sharding (painful) or NoSQL
   > 100K writes/sec → NoSQL (Cassandra, DynamoDB)
```

### Secondary Considerations

| Factor | Favor SQL | Favor NoSQL |
|---|---|---|
| Schema stability | Known upfront | Changes frequently |
| Query flexibility | Need ad-hoc queries | Access patterns are fixed |
| Consistency requirement | Strong (ACID) | Eventual is acceptable |
| Team familiarity | SQL expertise | NoSQL expertise |
| Operational complexity | Prefer simplicity | Can manage distributed system |
| Data relationships | Many foreign keys | Few/no relationships |
| Write pattern | Moderate, transactional | High volume, append-heavy |

### The Polyglot Persistence Pattern

In practice, large systems use multiple databases — each for what it does best:

```
User Service
  └─ PostgreSQL — user accounts, profiles, relationships (ACID, relational)

Post Service
  └─ Cassandra — posts and feeds (high write throughput, time-ordered)
  └─ Elasticsearch — post search (full-text, relevance ranking)

Session Service
  └─ Redis — sessions, rate limiting, ephemeral data (sub-millisecond KV)

Analytics Service
  └─ Kafka → ClickHouse — event streams, aggregations (columnar OLAP)

Media Service
  └─ S3 — blob storage (images, videos)
  └─ PostgreSQL — metadata (file names, owners, sizes)
```

**Interview tip:** Proposing polyglot persistence shows maturity. You're not picking one database for everything — you're picking the right tool for each data access pattern.

---

## Common Interview Scenarios

### Scenario 1: Design Twitter/Instagram

```
Data: Users, posts, follows, likes, comments, DMs

SQL (PostgreSQL):
  - Users, follows, DMs — relational, ACID needed
  - Good for < 10M users

Add as you scale:
  - Redis → cache hot profiles, rate limiting, session storage
  - Cassandra → feed timeline (high write throughput, partition by user_id)
  - Elasticsearch → post/user search
  - CDN + S3 → media storage

Decision: Start SQL, add NoSQL for specific bottlenecks.
Don't: Put everything in MongoDB from day one "because it scales."
```

### Scenario 2: Design Uber (Ride Matching)

```
Data: Drivers, riders, trips, locations, payments

Location data (driver positions updated every 5 seconds):
  → Redis Geospatial — sub-millisecond lookup of nearby drivers
  → Not SQL — can't handle this write rate with geospatial queries

Trip data:
  → PostgreSQL — ACID transactions (payment + trip creation must be atomic)

Historical trip data (analytics):
  → Cassandra or BigQuery — partition by driver_id/rider_id, time-ordered

Decision: Redis for real-time location, PostgreSQL for transactions, Cassandra for history.
```

### Scenario 3: Design a Rate Limiter

```
Need: Check and increment request count per user per minute

Redis (correct choice):
  INCR rate:user:42:202401011015   ← atomic increment
  EXPIRE rate:user:42:202401011015 60

SQL (wrong choice):
  BEGIN;
  SELECT count FROM rate_limits WHERE user_id=42 AND window='202401011015';
  UPDATE rate_limits SET count = count + 1 ...;
  COMMIT;
  → Too slow, too much contention, doesn't scale

Decision: Redis is purpose-built for this. Redis INCR is atomic, sub-millisecond.
```

### Scenario 4: Design a Metrics/Monitoring System

```
Need: Ingest millions of data points/sec, query time-series aggregations

Cassandra or TimescaleDB (PostgreSQL extension):
  - Partition by metric_name, cluster by timestamp
  - Efficient time-range scans
  - Compresses time-series data well

ClickHouse (columnar):
  - Blazing fast aggregations (AVG, P99, COUNT over billions of rows)
  - Columnar storage reads only columns needed

SQL (PostgreSQL alone, wrong choice):
  - Can't handle the write throughput
  - Row-oriented storage is inefficient for aggregations over billions of rows

Decision: Cassandra for ingestion, ClickHouse for aggregation queries.
```

### Scenario 5: Design an E-Commerce Platform

```
Products:
  → MongoDB — flexible schema (laptop has CPU/RAM, shirt has size/color)
  → Elasticsearch — product search with faceted filters

Orders and Payments:
  → PostgreSQL — ACID is non-negotiable
  → "Charge card + create order + decrement inventory" must be atomic

Inventory (flash sale — thousands of concurrent purchases):
  → PostgreSQL with SELECT FOR UPDATE (pessimistic lock)
  → Or Redis atomic DECR + async sync to PostgreSQL

Sessions and Cart:
  → Redis — fast, ephemeral, no durability needed

Decision: Polyglot — right database for each domain.
```

---

## Convincing Interview Phrases

**Choosing SQL:**
> *"The data has natural relationships — users, orders, and products reference each other. JOINs are the clean way to query this, and ACID transactions ensure we never have a payment without a corresponding order. PostgreSQL handles this well up to hundreds of millions of rows with proper indexing and read replicas."*

**Choosing NoSQL (Cassandra):**
> *"We're looking at 50M events per day — IoT sensor readings, always queried by device ID in time order. The access pattern is fixed, there are no joins needed, and we need horizontal write scale. Cassandra's partition-by-device, cluster-by-time model is purpose-built for this."*

**Choosing NoSQL (Redis):**
> *"For session storage and rate limiting, Redis is the right call. It's sub-millisecond, INCR is atomic, and TTL-based expiry is built in. There's no relational structure here — it's pure key-value."*

**Polyglot (most mature answer):**
> *"I wouldn't use a single database. PostgreSQL for the transactional core — users, orders, payments — because ACID is non-negotiable there. Cassandra for the event feed because the write volume is too high for a single SQL primary. Redis for sessions and hot caches. Elasticsearch for search. Each database is doing what it was designed for."*

**When pushed "why not just use NoSQL for everything?":**
> *"NoSQL gives up joins and cross-partition transactions. That's fine when your access patterns are simple and fixed. But for the order management domain, I need to atomically update inventory, create an order, and process payment across three tables. Without ACID, I risk charging a user and failing to create the order. That's a real-world incident. PostgreSQL's consistency guarantee is worth the scaling trade-off here."*

**When pushed "why not just use SQL for everything?":**
> *"SQL can scale reads easily with replicas. But scaling writes past a single node requires manual sharding — and once you shard, you lose JOINs and foreign keys. At that point you've given up the relational model anyway, so you might as well use a database designed for it. For the feed service, the write volume and access pattern are a perfect fit for Cassandra — no joins needed, always queried by user ID."*

---

## Numbers to Know

| Database | Write Throughput | Read Throughput | Consistency | Latency |
|---|---|---|---|---|
| PostgreSQL (single node) | 5K–10K/sec | 50K–100K/sec | Strong (ACID) | 1–10ms |
| PostgreSQL (+ replicas) | 5K–10K/sec | 200K–500K/sec | Strong (primary), Eventual (replicas) | 1–10ms |
| Cassandra (10 nodes) | 500K–1M/sec | 500K–1M/sec | Tunable (eventual to strong) | 1–5ms |
| DynamoDB | Effectively unlimited | Effectively unlimited | Eventual (default) / Strong (opt-in) | < 10ms |
| Redis | 100K–1M/sec | 100K–1M/sec | Strong (single node) | < 1ms |
| MongoDB | 50K–100K/sec | 100K–200K/sec | Eventual (default) | 1–10ms |
| Elasticsearch | 10K–50K/sec | 10K–100K/sec | Eventual | 10–100ms |

> These are ballpark figures for interview discussions, not benchmarks. Real numbers depend heavily on hardware, data size, query complexity, and configuration.

---

## Summary Cheat Sheet

```
SQL (PostgreSQL)
────────────────────────────────────────────────────────
Strengths:    ACID, JOINs, flexible queries, referential integrity
Weaknesses:   Hard to scale writes horizontally, schema migrations
Scale:        Reads → easy (replicas). Writes → painful (sharding)
Choose when:  Relational data, ACID required, unknown query patterns
Avoid when:   Millions of writes/sec, multi-region active-active


NoSQL — Key-Value (Redis)
────────────────────────────────────────────────────────
Strengths:    Sub-millisecond, atomic ops, TTL, pub/sub
Weaknesses:   Data must fit in memory, no complex queries
Choose when:  Sessions, caching, rate limiting, leaderboards


NoSQL — Document (MongoDB)
────────────────────────────────────────────────────────
Strengths:    Flexible schema, natural JSON storage, horizontal scale
Weaknesses:   Weak cross-document transactions, no JOINs
Choose when:  Flexible/evolving schema, self-contained documents


NoSQL — Wide-Column (Cassandra)
────────────────────────────────────────────────────────
Strengths:    Massive write throughput, linear horizontal scale, HA
Weaknesses:   No JOINs, must know access patterns upfront, eventual consistency
Choose when:  Time-series, feeds, IoT, high write volume with fixed access patterns


NoSQL — Graph (Neo4j)
────────────────────────────────────────────────────────
Strengths:    Relationship traversal, graph algorithms
Weaknesses:   Niche, hard to scale writes
Choose when:  Social graph, recommendations, fraud detection


Search (Elasticsearch)
────────────────────────────────────────────────────────
Strengths:    Full-text search, relevance ranking, faceting, aggregations
Weaknesses:   Eventual consistency, not a primary store, complex ops
Choose when:  Product search, log analysis, auto-complete


The Mature Interview Answer
────────────────────────────────────────────────────────
"Use PostgreSQL as the default. Add specialized databases only when
 you hit a specific bottleneck or have a clear access pattern mismatch.
 PostgreSQL for transactions. Redis for caching and ephemeral KV.
 Cassandra when write volume exceeds single-node SQL capacity.
 Elasticsearch for search. Each database does one thing well."
```

---

## Quick Reference: The Five Questions to Ask

Before picking a database in an interview, answer these five questions:

```
1. RELATIONSHIPS
   Are there JOINs between entities?
   YES → lean SQL | NO → lean NoSQL

2. CONSISTENCY
   Is ACID required? Can you tolerate eventual consistency?
   ACID required → SQL | Eventual OK → NoSQL

3. SCALE
   What is the write throughput requirement?
   < 10K/sec → SQL | > 100K/sec → NoSQL

4. ACCESS PATTERNS
   Do you know exactly how data will be queried?
   FIXED patterns → NoSQL (optimize for them)
   UNKNOWN patterns → SQL (query flexibility)

5. SCHEMA
   Is the data structure stable?
   STABLE → SQL (schema enforcement is a feature)
   EVOLVING rapidly → NoSQL (schema flexibility)
```

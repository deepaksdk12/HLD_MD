# PostgreSQL — System Design Deep Dive

> **Goal:** Make informed architectural decisions about PostgreSQL — when to use it, when not to, and what trade-offs matter in an interview.

---

## Table of Contents

1. [A Motivating Example](#a-motivating-example)
2. [Core Capabilities & Limitations](#core-capabilities--limitations)
   - [Read Performance](#read-performance)
   - [Write Performance](#write-performance)
   - [Replication](#replication)
   - [Data Consistency](#data-consistency)
3. [When to Use PostgreSQL](#when-to-use-postgresql)
4. [When to Consider Alternatives](#when-to-consider-alternatives)
5. [SQL vs NoSQL Decision Framework](#sql-vs-nosql-decision-framework)
6. [Common Interview Scenarios](#common-interview-scenarios)
7. [Numbers to Know](#numbers-to-know)
8. [Summary Cheat Sheet](#summary-cheat-sheet)
9. [Appendix: SQL & ACID Refresher](#appendix-sql--acid-refresher)

---

## A Motivating Example

Imagine designing a **social media platform** — not Facebook-scale, but one that is growing. It needs to handle:

- Users create posts
- Users comment on posts
- Users follow other users
- Users like posts and comments
- Users send direct messages (DMs)

### Why this is interesting

| Operation | Requirement |
|---|---|
| Creating a DM thread | **Atomic** — create thread + add participants + store first message together |
| Comments, follows | **Referential integrity** — can't comment on a deleted post |
| Like counts | **Eventually consistent** — slight delay is acceptable |
| Profile page | **Efficient read** — recent posts + follower count in one round trip |
| Search users/posts | **Full-text search** capability |
| Growing data | **Scalability** — indexes, partitioning, replication |

This mix of relational data, mixed consistency needs, search, and growth pressure makes it a perfect PostgreSQL case study.

---

## Core Capabilities & Limitations

### Read Performance

In most applications, **reads vastly outnumber writes** (10:1 to 100:1 ratio). Read optimization is usually the first lever to pull.

#### B-tree Indexes (Default)

Best for:
- Exact matches: `WHERE email = 'user@example.com'`
- Range queries: `WHERE created_at > '2024-01-01'`
- Sorting: `ORDER BY username` (when ORDER BY columns match index column order)

```sql
-- Single-column index
CREATE INDEX idx_users_email ON users(email);

-- Multi-column index for common query patterns
-- Useful when queries filter by user_id AND sort/filter by created_at
CREATE INDEX idx_posts_user_date ON posts(user_id, created_at DESC);
```

**Interview trap:** Don't suggest indexing every column.
Each index:
- Slows down writes (index must be updated on every INSERT/UPDATE/DELETE)
- Consumes disk space
- May be ignored by the query planner if a sequential scan is cheaper

#### Partial Indexes

Index only a subset of rows — useful when most queries target a specific value:

```sql
-- Only index active users, not soft-deleted ones
CREATE INDEX idx_active_users ON users(email)
WHERE deleted_at IS NULL;
```

#### Hash Indexes

Faster than B-tree for **exact equality lookups only** (no range queries):

```sql
CREATE INDEX idx_sessions_token ON sessions USING HASH (token);
```

#### GIN Indexes (for JSONB and Full-Text Search)

```sql
-- For full-text search on post content
CREATE INDEX idx_posts_fts ON posts USING GIN (to_tsvector('english', content));

-- For querying inside JSONB columns
CREATE INDEX idx_users_metadata ON users USING GIN (metadata);
```

#### Query Optimization Essentials

**EXPLAIN ANALYZE** — your best friend for understanding query performance:

```sql
EXPLAIN ANALYZE
SELECT p.*, u.username
FROM posts p
JOIN users u ON p.user_id = u.id
WHERE p.created_at > NOW() - INTERVAL '7 days'
ORDER BY p.created_at DESC
LIMIT 20;
```

Key patterns to watch for:
- **Seq Scan** on large tables → add an index
- **Nested Loop** on large joins → consider hash joins or denormalization
- **High row estimates vs actual** → run `ANALYZE` to update statistics

**N+1 Query Problem** — a classic interview red flag:

```sql
-- BAD: 1 query for posts + N queries for each user
SELECT * FROM posts LIMIT 10;
-- then for each post: SELECT * FROM users WHERE id = ?

-- GOOD: 1 query with JOIN
SELECT p.*, u.username, u.avatar_url
FROM posts p
JOIN users u ON p.user_id = u.id
LIMIT 10;
```

**Covering Indexes** — include all columns the query needs so PostgreSQL never touches the table:

```sql
-- Query: SELECT user_id, created_at FROM posts WHERE user_id = 123
-- This index "covers" the query — no table heap access needed
CREATE INDEX idx_posts_covering ON posts(user_id) INCLUDE (created_at, title);
```

---

### Write Performance

#### Throughput Limitations

A well-tuned single PostgreSQL instance can handle roughly:
- **~5,000–10,000 writes/sec** for simple inserts
- **~1,000–5,000 writes/sec** for complex transactions with multiple tables
- Drops significantly with many indexes and concurrent writes

This is plenty for most applications but becomes a bottleneck at massive scale (think millions of events/sec).

#### Why Writes Are Slower Than Reads

Every write in PostgreSQL must:
1. Write to the **Write-Ahead Log (WAL)** for durability
2. Update the actual table data
3. Update **every index** on the table
4. Handle **MVCC** (keep old row versions for concurrent readers)

#### Write Performance Optimizations

**Batch inserts** instead of individual inserts:

```sql
-- BAD: 1000 round trips
INSERT INTO events (user_id, type) VALUES (1, 'click');
INSERT INTO events (user_id, type) VALUES (2, 'view');
-- ... repeated 1000 times

-- GOOD: 1 round trip
INSERT INTO events (user_id, type) VALUES
  (1, 'click'),
  (2, 'view'),
  -- ... all 1000 rows
  (1000, 'click');
```

**COPY command** for bulk loads (10–100x faster than INSERT):

```sql
COPY events (user_id, type, created_at)
FROM '/tmp/events.csv'
WITH (FORMAT csv, HEADER true);
```

**Async writes for non-critical data:**

```sql
-- Disable fsync for a session (dangerous — use only for non-critical bulk loads)
SET synchronous_commit = OFF;
```

**Table Partitioning** — split a huge table into smaller physical pieces:

```sql
-- Partition posts by month
CREATE TABLE posts (
    id BIGSERIAL,
    user_id BIGINT,
    content TEXT,
    created_at TIMESTAMPTZ NOT NULL
) PARTITION BY RANGE (created_at);

CREATE TABLE posts_2024_01 PARTITION OF posts
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

CREATE TABLE posts_2024_02 PARTITION OF posts
    FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');
```

Benefits:
- Queries that filter by date only scan the relevant partition
- Old partitions can be archived or dropped cheaply
- Indexes on each partition are smaller and faster

---

### Replication

#### Primary-Replica (Read Replicas)

The most common PostgreSQL scaling pattern:

```
        Writes
        ──────►  [ Primary ]
                      │
          Streaming   │  Replication
          ────────────┼──────────────
                      ▼
              [ Replica 1 ]  ◄── Reads
              [ Replica 2 ]  ◄── Reads
              [ Replica 3 ]  ◄── Reads
```

- **Synchronous replication:** Primary waits for at least one replica to confirm before acknowledging the write. Zero data loss, but higher write latency.
- **Asynchronous replication:** Primary does not wait. Lower write latency, but replicas may lag — risk of losing recent data if primary crashes.

**Interview tip:** For our social media platform, route profile reads and post browsing to replicas. Route writes and DM reads (where consistency matters) to the primary.

#### Replication Lag

Replicas are **eventually consistent** with the primary. In practice, lag is typically < 100ms on a healthy cluster, but can grow under heavy write load.

This matters when:
- A user posts something and immediately refreshes — they might not see their own post if the read hits a lagging replica
- **Solution:** Route a user's own writes/reads to the primary for a short window (called "read-your-writes" consistency)

#### High Availability (HA)

Use a tool like **Patroni** or managed services (AWS RDS, Cloud SQL) for automatic failover:

```
Primary fails ──► Patroni promotes a replica to primary ──► ~30s downtime
```

For zero-downtime scenarios, use connection poolers like **PgBouncer** in front of your cluster.

---

### Data Consistency

#### ACID Properties

PostgreSQL is fully ACID-compliant — this is one of its biggest selling points.

| Property | What it means | Social media example |
|---|---|---|
| **Atomicity** | All or nothing — partial writes never persist | Creating a DM: thread + participants + first message all succeed or all roll back |
| **Consistency** | DB moves from one valid state to another | A follow can't reference a non-existent user |
| **Isolation** | Concurrent transactions don't interfere | Two users liking a post simultaneously both register correctly |
| **Durability** | Committed data survives crashes | A sent DM is never lost even if the server crashes immediately after |

#### Transactions

```sql
BEGIN;

-- Deduct from sender
UPDATE wallets SET balance = balance - 100 WHERE user_id = 1;

-- Credit to receiver
UPDATE wallets SET balance = balance + 100 WHERE user_id = 2;

-- Only commits if both succeed
COMMIT;

-- Rolls back both if anything fails
ROLLBACK;
```

#### Isolation Levels

PostgreSQL supports four isolation levels. Know these for interviews:

| Level | Prevents | Use when |
|---|---|---|
| **Read Committed** (default) | Dirty reads | Most OLTP workloads |
| **Repeatable Read** | Dirty + non-repeatable reads | Reports that must see consistent snapshot |
| **Serializable** | All anomalies | Financial transactions, inventory |

```sql
-- Serializable transaction — behaves as if transactions ran one at a time
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
SELECT balance FROM wallets WHERE user_id = 1;
UPDATE wallets SET balance = balance - 100 WHERE user_id = 1;
COMMIT;
```

#### Constraints for Referential Integrity

```sql
CREATE TABLE posts (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    content TEXT NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE comments (
    id BIGSERIAL PRIMARY KEY,
    post_id BIGINT NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
    user_id BIGINT NOT NULL REFERENCES users(id),
    content TEXT NOT NULL
);
```

`ON DELETE CASCADE` ensures comments are deleted when a post is deleted — no orphaned rows.

#### Optimistic vs Pessimistic Locking

**Pessimistic locking** — lock the row while you work on it:

```sql
-- No other transaction can modify this row until we COMMIT
SELECT * FROM inventory WHERE product_id = 42 FOR UPDATE;
UPDATE inventory SET quantity = quantity - 1 WHERE product_id = 42;
COMMIT;
```

**Optimistic locking** — check that nothing changed before committing (better for low-contention scenarios):

```sql
-- Use a version column
UPDATE posts
SET content = 'updated', version = version + 1
WHERE id = 123 AND version = 5;  -- fails if someone else updated it first
```

---

## When to Use PostgreSQL

Choose PostgreSQL when your system has:

### 1. Complex Relationships Between Entities
Your data has clear foreign key relationships and JOINs are the natural way to query it. E-commerce (users → orders → products), social platforms (users → posts → comments), SaaS apps.

### 2. ACID Transactions Are Non-Negotiable
Payments, financial ledgers, inventory management, booking systems — anywhere partial writes cause real-world damage.

**Convincing phrase:** *"We can't afford inconsistency here — if a payment deducts from one account but doesn't credit another, we have a production incident on our hands."*

### 3. Schema Is Stable and Well-Understood
Requirements are clear enough that you know the shape of your data. Schema changes in PostgreSQL require migrations, which are manageable but not free.

### 4. Complex Queries and Reporting
Ad-hoc analytics, aggregations, GROUP BY, window functions. PostgreSQL's SQL is far more expressive than most NoSQL query languages.

```sql
-- Top 5 users by post engagement this week
SELECT
    u.username,
    COUNT(DISTINCT p.id) AS posts,
    SUM(l.like_count) AS total_likes,
    RANK() OVER (ORDER BY SUM(l.like_count) DESC) AS rank
FROM users u
JOIN posts p ON p.user_id = u.id
LEFT JOIN (
    SELECT post_id, COUNT(*) AS like_count FROM likes GROUP BY post_id
) l ON l.post_id = p.id
WHERE p.created_at > NOW() - INTERVAL '7 days'
GROUP BY u.id, u.username
ORDER BY total_likes DESC
LIMIT 5;
```

### 5. You Need Both Relational and Document Storage
PostgreSQL's `JSONB` column type gives you flexible document storage without leaving the relational world:

```sql
-- Store flexible product attributes without schema changes
CREATE TABLE products (
    id BIGSERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    price NUMERIC(10,2),
    attributes JSONB  -- {"color": "red", "size": "M", "material": "cotton"}
);

-- Query inside JSONB
SELECT * FROM products WHERE attributes->>'color' = 'red';

-- Index JSONB for performance
CREATE INDEX idx_products_attrs ON products USING GIN (attributes);
```

---

## When to Consider Alternatives

### Switch to Cassandra / DynamoDB when:
- You need **massive horizontal write scalability** (millions of events/sec — IoT, activity logs, time-series)
- Your access pattern is always **by a single partition key** (no complex joins ever)
- You need **multi-region active-active writes** with no single primary

**Convincing phrase:** *"We know exactly how we'll always query this — by user ID. There are no joins. At 50M writes/day, Cassandra's linear horizontal scaling is the right fit, not a replicated PostgreSQL primary."*

### Switch to Elasticsearch when:
- You need **full-text search** at scale with relevance ranking, fuzzy matching, faceted filters
- PostgreSQL's `tsvector` full-text search works for basic cases, but Elasticsearch is built for search as the primary use case

### Switch to Redis when:
- You need **sub-millisecond key-value lookups** (sessions, rate limiting, leaderboards)
- Data fits in memory and durability is secondary

### Stay with PostgreSQL but add a cache (Redis) when:
- Your read replicas are still not fast enough for hot data
- You have a high-traffic read path with a stable result (user profile, product detail)

### Common interview mistake
> "NoSQL scales better than PostgreSQL."

This is too vague. PostgreSQL with read replicas, connection pooling, and partitioning scales to **hundreds of millions of rows and thousands of QPS**. Instagram ran on PostgreSQL. Reddit runs on PostgreSQL. The real question is *which specific bottleneck* are you hitting.

---

## SQL vs NoSQL Decision Framework

```
Is your data highly relational with JOINs?
    YES → Strong signal for SQL (PostgreSQL)
    NO  ↓

Do you need ACID transactions across multiple entities?
    YES → Strong signal for SQL (PostgreSQL)
    NO  ↓

Is horizontal write scale the primary constraint? (millions/sec)
    YES → NoSQL (Cassandra, DynamoDB)
    NO  ↓

Is your schema highly dynamic / document-shaped?
    YES → NoSQL (MongoDB) or PostgreSQL + JSONB
    NO  ↓

Default → PostgreSQL
```

**Strong middle-ground interview answer:**
> *"I'd start with PostgreSQL. It gives me ACID, handles JOINs, supports JSONB for flexible fields, and scales well with read replicas. I'd only move to NoSQL if I hit a specific bottleneck — like needing multi-region active-active writes or 10M+ events/sec with a single partition-key access pattern."*

---

## Common Interview Scenarios

### Social Media Platform (Instagram-like)
- **Users, posts, comments, follows:** PostgreSQL — relational, ACID, complex queries
- **Like counts:** PostgreSQL counter columns with eventual consistency, or Redis for hot counters
- **Feed generation:** PostgreSQL for small-scale; pre-computed feed in Redis/Cassandra at scale
- **Search:** PostgreSQL `tsvector` for basic search; Elasticsearch for advanced relevance

### Payment / Financial System
- **Ledger, wallets, transactions:** PostgreSQL — ACID is non-negotiable
- **Fraud signals (high write volume):** Kafka → Cassandra for event storage
- **Reporting/analytics:** Read replica or separate OLAP (Redshift, BigQuery)

### E-Commerce
- **Products, orders, users:** PostgreSQL
- **Inventory with concurrent purchases:** PostgreSQL with `SELECT FOR UPDATE` or serializable transactions
- **Product catalog search:** Elasticsearch
- **Session/cart:** Redis

### Real-Time Analytics / Metrics
- **Raw event ingestion:** Kafka → Cassandra or TimescaleDB (PostgreSQL extension for time-series)
- **Aggregated dashboards:** PostgreSQL read replica or OLAP warehouse

---

## Numbers to Know

| Metric | Value | Notes |
|---|---|---|
| Single instance write throughput | ~5K–10K writes/sec | Simple inserts, well-tuned |
| Single instance read throughput | ~50K–100K reads/sec | With proper indexes and caching |
| Replication lag (healthy cluster) | < 100ms | Can grow under heavy write load |
| Max practical table size | Hundreds of GB to low TB | Beyond this, consider partitioning or sharding |
| Failover time (with Patroni) | ~30 seconds | For automatic primary promotion |
| Connection overhead | ~5–10MB per connection | Use PgBouncer connection pooler in production |
| Index write overhead | ~10–30% write slowdown | Per additional index on a table |

---

## Summary Cheat Sheet

```
PostgreSQL Strengths
├── ACID transactions (atomicity, consistency, isolation, durability)
├── Complex JOINs across related entities
├── Rich indexing (B-tree, Hash, GIN, partial, covering)
├── JSONB for flexible document storage alongside relational data
├── Full-text search (basic-to-moderate scale)
├── Window functions, CTEs, advanced SQL
├── Read scaling via streaming replication + read replicas
└── Mature ecosystem (Patroni HA, PgBouncer pooling, pgvector, TimescaleDB)

PostgreSQL Limitations
├── Write throughput ceiling (~10K/sec single node)
├── No native horizontal write sharding
├── Schema migrations require careful planning
├── Replication is eventually consistent (replica lag)
└── Not designed for sub-millisecond key-value lookups

When to choose PostgreSQL
├── Relational data with JOINs
├── ACID required (payments, bookings, inventory)
├── Complex queries / reporting
└── Mixed relational + document data (JSONB)

When to look elsewhere
├── Millions of writes/sec → Cassandra / DynamoDB
├── Advanced full-text search → Elasticsearch
├── Sub-millisecond key-value → Redis
└── Columnar analytics → Redshift / BigQuery / ClickHouse
```

---

## Appendix: SQL & ACID Refresher

### Relational Database Principles

- Data is organized into **tables** (rows and columns)
- Tables relate to each other via **foreign keys**
- **Normalization** reduces duplication by splitting data into separate tables and joining at query time
- **Denormalization** duplicates data to avoid joins — useful for read performance at the cost of write complexity

### ACID Properties

#### Atomicity (All or Nothing)
A transaction is treated as a single unit. Either all statements succeed and are committed, or none are. There is no partial commit.

```sql
BEGIN;
INSERT INTO orders (user_id, total) VALUES (1, 99.99);
INSERT INTO order_items (order_id, product_id, qty) VALUES (LASTVAL(), 42, 1);
UPDATE inventory SET quantity = quantity - 1 WHERE product_id = 42;
COMMIT;  -- All three succeed together, or none do
```

#### Consistency (Data Integrity)
Every transaction brings the database from one valid state to another. Constraints (NOT NULL, UNIQUE, FOREIGN KEY, CHECK) are enforced at commit time.

#### Isolation (Concurrent Transactions)
Multiple transactions running simultaneously do not see each other's intermediate (uncommitted) state. PostgreSQL uses MVCC (Multi-Version Concurrency Control) — readers don't block writers and writers don't block readers.

#### Durability (Permanent Storage)
Once a transaction commits, it is permanent — even if the server crashes immediately after. PostgreSQL achieves this via the Write-Ahead Log (WAL): every change is written to the WAL before the transaction is acknowledged.

### Why ACID Matters in Interviews

When an interviewer asks "how do you handle consistency?" or "what happens if the server crashes mid-transaction?", the answer for PostgreSQL is:
> *"PostgreSQL's ACID guarantees handle this. The WAL ensures durability, foreign keys enforce referential integrity, and transactions ensure atomicity — a partial write is impossible."*

### Essential SQL Commands

```sql
-- DDL: Define structure
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    email TEXT UNIQUE NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- DML: Manipulate data
INSERT INTO users (email) VALUES ('alice@example.com');
UPDATE users SET email = 'new@example.com' WHERE id = 1;
DELETE FROM users WHERE id = 1;

-- DQL: Query data
SELECT u.email, COUNT(p.id) AS post_count
FROM users u
LEFT JOIN posts p ON p.user_id = u.id
GROUP BY u.id, u.email
HAVING COUNT(p.id) > 5
ORDER BY post_count DESC
LIMIT 10;

-- Window functions
SELECT
    user_id,
    created_at,
    ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY created_at DESC) AS rn
FROM posts;

-- CTE (Common Table Expression)
WITH recent_posts AS (
    SELECT * FROM posts WHERE created_at > NOW() - INTERVAL '24 hours'
)
SELECT u.username, COUNT(*) AS posts_today
FROM recent_posts rp
JOIN users u ON u.id = rp.user_id
GROUP BY u.username
ORDER BY posts_today DESC;
```

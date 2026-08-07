si# Database Indexing

## Why Indexes Exist

Without an index, a database must do a **full table scan** — checking every row sequentially to find a match. For millions of rows, this is painfully slow.

An index is a separate data structure that maps column values to row locations, letting the database jump directly to the relevant rows instead of scanning everything.

**Cost of indexes:**
- Extra disk space (can be nearly as large as the table itself)
- Slower writes — every INSERT/UPDATE must update all indexes on the table
- Not always worth it: small tables, write-heavy tables with rare reads, low-cardinality columns (e.g. boolean flags)

---

## B-Tree Index (Default)

The most common index type. Used by PostgreSQL, MySQL, MongoDB for nearly all standard indexes.

### Structure

A self-balancing tree where:
- Each node holds multiple sorted keys and pointers to children
- All leaf nodes are at the same depth
- Node size matches a disk page (~8KB) to minimize I/O
- Finding any record requires traversing root → internal nodes → leaf

```
              [300 | 600]
             /     |      \
      [100|200] [400|500] [700|800]
       /   \      / \       / \
    leaf  leaf  leaf leaf  leaf leaf
```

### Properties

| Property | Detail |
|---|---|
| Time complexity | O(log n) for reads, inserts, deletes |
| Supports range queries | YES — tree is sorted |
| Supports ORDER BY | YES |
| Supports equality | YES |
| Self-balancing | YES — no performance cliffs on random inserts |

### When PostgreSQL creates B-Trees automatically

```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,     -- B-tree index auto-created
    email VARCHAR(255) UNIQUE  -- B-tree index auto-created
);
```

### Use B-Tree when

- Default choice for most columns
- Range queries: `age > 25`, `created_at BETWEEN ...`
- Equality lookups: `email = 'x@y.com'`
- Sorting: `ORDER BY last_name`
- You need a safe, general-purpose index

---

## LSM Tree (Log-Structured Merge Tree)

Designed for **write-heavy workloads** where B-trees become a bottleneck.

### How it works

Writes go to an in-memory buffer (MemTable) first — no immediate disk write. When the buffer fills, it's flushed to disk as an immutable sorted file (SSTable). Background compaction merges SSTables periodically.

```
Write → MemTable (in-memory)
           ↓ (when full)
        SSTable L0 (disk)
           ↓ (compaction)
        SSTable L1 → L2 → L3 (merged, sorted)
```

### Trade-offs

| | B-Tree | LSM Tree |
|---|---|---|
| Write speed | Moderate | Very fast (sequential) |
| Read speed | Fast | Slower (may check multiple SSTables) |
| Space amplification | Low | Higher (before compaction) |
| Write amplification | Higher | Lower |

### Read problem fix — Bloom Filters

Before checking each SSTable for a key, a **Bloom filter** answers "definitely not here" in O(1), skipping irrelevant files.

### Used by

- **Cassandra** — primary storage engine
- **RocksDB** — used internally by many systems (MySQL MyRocks, TiKV)
- **DynamoDB** — LSM-style internals (not publicly documented but widely understood)
- **HBase, LevelDB**

### Use LSM when

- Ingesting millions of events/sec (metrics, logs, IoT)
- Time-series data with append-only writes
- Write throughput matters more than read latency

---

## Hash Index

Uses a hash function to map column values directly to row locations.

### How it works

```
email → hash("alice@x.com") → bucket 42 → row pointer
```

Lookup is O(1) — no tree traversal needed.

### Limitations

| Capability | Hash Index |
|---|---|
| Equality: `email = 'x'` | YES — O(1) |
| Range: `age > 25` | NO — hash destroys order |
| ORDER BY | NO |
| LIKE prefix | NO |

PostgreSQL supports hash indexes but B-trees are still preferred because B-trees handle both equality and range queries.

### Use Hash when

- Pure key-value lookups with no range queries
- In-memory hash tables (Redis)
- Join operations (hash join in query planners)

---

## Geospatial Index

Standard B-trees can't handle 2D spatial queries like "find all restaurants within 5km of me" efficiently.

### Geohash

Encodes (latitude, longitude) into a single string. Nearby locations share a common prefix.

```
(37.7749, -122.4194)  →  "9q8yy"
(37.7751, -122.4190)  →  "9q8yy"  (same prefix = nearby)
```

Store geohash as a regular indexed string column. Range query on prefix finds nearby points.

**Problem:** Grid boundary issue — two points close in reality can have very different geohash prefixes if they straddle a grid boundary. Fix: query multiple adjacent cells.

### Quadtree

Recursively divides 2D space into 4 quadrants. Each node subdivides until it contains ≤ N points.

```
+--------+--------+
|        |   | x  |
|        |---+----|
|        |   |    |
+--------+--------+
|        |        |
|        |        |
+--------+--------+
```

Good for unevenly distributed data (cities are dense, oceans are empty). Used by Twitter for geo-search.

### R-Tree

Groups nearby objects into minimum bounding rectangles. Each parent node covers the bounding box of all its children.

Used by PostGIS (PostgreSQL spatial extension) and most SQL databases with geospatial support.

```sql
-- PostGIS example
CREATE INDEX ON restaurants USING GIST(location);

SELECT name FROM restaurants
WHERE ST_DWithin(location, ST_MakePoint(-122.41, 37.77)::geography, 5000);
-- finds all within 5km
```

### Use Geospatial index when

- "Nearby" queries: restaurants, drivers, venues
- PostGIS / MySQL Spatial: R-tree via GIST index
- Cassandra / DynamoDB: geohash as partition/sort key

---

## Inverted Index

Used for **full-text search**. Maps each word to the list of documents containing it.

```
"database" → [doc1, doc3, doc7]
"index"    → [doc1, doc2, doc5]

Query: "database AND index" → intersection → [doc1]
```

### How Elasticsearch builds it

```
Document: "Fast database indexing guide"
Tokens:   [fast, database, index, guide]  (after stemming/lowercasing)

Inverted index:
  fast     → [doc_id, position]
  database → [doc_id, position]
  index    → [doc_id, position]
  guide    → [doc_id, position]
```

### Use Inverted Index when

- Full-text search: product names, article content, descriptions
- Autocomplete
- Tag or keyword filtering at scale
- Tools: Elasticsearch, OpenSearch, PostgreSQL `tsvector`

```sql
-- PostgreSQL full-text search with inverted index (GIN)
CREATE INDEX ON articles USING GIN(to_tsvector('english', content));

SELECT title FROM articles
WHERE to_tsvector('english', content) @@ to_tsquery('database & index');
```

---

## Index Optimization Patterns

### Composite Index

An index on multiple columns together.

```sql
CREATE INDEX idx_user_status ON bookings(user_id, status);
```

**Column order matters critically.**

```sql
-- Uses index (user_id is leftmost)
SELECT * FROM bookings WHERE user_id = 123;
SELECT * FROM bookings WHERE user_id = 123 AND status = 'CONFIRMED';

-- Cannot use index (skips user_id)
SELECT * FROM bookings WHERE status = 'CONFIRMED';
```

Rule: put the **most selective** or **most queried** column first. Equality columns before range columns.

```sql
-- Good: equality first, range last
CREATE INDEX ON events(category, event_date);
WHERE category = 'music' AND event_date > '2025-01-01'

-- Bad: range first blocks use of category
CREATE INDEX ON events(event_date, category);
```

### Covering Index

An index that contains all columns a query needs — the database never touches the main table.

```sql
-- Query needs user_id, status, created_at
SELECT status, created_at FROM bookings WHERE user_id = 123;

-- Covering index: includes all three columns
CREATE INDEX idx_covering ON bookings(user_id, status, created_at);
-- DB reads only the index, never the heap — significant speedup
```

### Partial Index

Index only a subset of rows matching a condition.

```sql
-- Only index active users — much smaller, faster
CREATE INDEX idx_active_users ON users(email) WHERE active = true;

-- Only index unpaid orders
CREATE INDEX idx_unpaid ON orders(created_at) WHERE status = 'UNPAID';
```

Useful when queries almost always filter on a specific value, and that value covers a small fraction of rows.

---

## Quick Reference: Which Index to Use

| Use Case | Index Type |
|---|---|
| Default / general purpose | B-Tree |
| Pure equality lookups (key-value) | Hash |
| Write-heavy (metrics, logs, time-series) | LSM Tree |
| Geospatial / nearby queries | Geohash / Quadtree / R-Tree |
| Full-text search | Inverted Index (Elasticsearch, GIN) |
| Queries on multiple columns | Composite B-Tree |
| Avoid table access entirely | Covering Index |
| Index a specific subset of rows | Partial Index |

---

## Interview Tips

- **Always justify** why you're adding an index — interviewer wants to hear the trade-off (write cost vs read gain)
- **B-Tree is the safe default** — state it explicitly and only deviate with a reason
- For **ticketing/booking**: index on `(event_id, status)` for seat availability queries
- For **geolocation** (Uber, Yelp): mention geohash or PostGIS R-tree
- For **search** (Twitter, LinkedIn): Elasticsearch with inverted index
- For **time-series** (metrics, logs): Cassandra/InfluxDB with LSM tree
- **Covering indexes** are a strong signal of depth — mention them when the interviewer asks about read optimization

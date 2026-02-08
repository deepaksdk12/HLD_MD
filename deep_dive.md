# System Design Pattern Library
*A comprehensive collection of patterns from 16+ FAANG system design problems*

**Last Updated:** November 27, 2025

---

## Table of Contents

1. [Pattern Overview](#pattern-overview)
2. [Dealing with Contention](#dealing-with-contention)
3. [Scaling Reads](#scaling-reads)
4. [Scaling Writes](#scaling-writes)
5. [Handling Large Blobs](#handling-large-blobs)
6. [Real-time Updates](#real-time-updates)
7. [Multi-step Processes & Workflows](#multi-step-processes--workflows)
8. [Unique ID Generation](#unique-id-generation)
9. [Geospatial & Location-based Services](#geospatial--location-based-services)
10. [Search & Full-text Indexing](#search--full-text-indexing)
11. [Data Structures for Scale](#data-structures-for-scale)
12. [Technology Stack Reference](#technology-stack-reference)
13. [Problem-to-Pattern Mapping](#problem-to-pattern-mapping)

---

## Pattern Overview

This document consolidates patterns from analyzing the following system design problems:

**Core Infrastructure:**
- Ticketmaster (ticket booking, high contention)
- Dropbox (file storage, large blobs)
- Bit.ly (URL shortening, read-heavy)
- GoPuff (local delivery, inventory management)

**Social & Communication:**
- FB News Feed (fan-out, pre-computation)
- WhatsApp (real-time messaging, bi-directional)
- FB Live Comments (real-time broadcasting)
- FB Post Search (full-text search)
- Tinder (matching, geospatial)

**Media & Content:**
- YouTube (video streaming, transcoding)
- LeetCode (code execution, isolation)
- Web Crawler (distributed processing)

**Data Processing:**
- YouTube Top K (stream processing, aggregation)
- Distributed Rate Limiter (atomic operations)
- Ad Click Aggregator (real-time analytics)
- Uber (ride matching, location updates)

---

## Dealing with Contention

**When to Use:** Multiple users/processes competing for the same resource (inventory, tickets, rides, etc.)

### Pattern: Distributed Locks with TTL

**Problem:** Prevent double-booking when multiple users try to reserve the same resource simultaneously.

**Solution:**
- Use Redis distributed lock (Redlock algorithm) with automatic expiration (TTL)
- Lock is acquired before checking/modifying shared resource
- TTL ensures locks don't remain stuck if process dies

**Implementation Details:**
```
1. User selects resource (seat, item, ride)
2. Service attempts to acquire lock in Redis:
   - SET lock_key unique_value NX EX 600  // 10 min TTL
3. If lock acquired:
   - Mark resource as "reserved" in DB
   - Return reservation ID to user
4. Lock auto-expires after TTL
5. On purchase confirmation, update status to "sold" and release lock
```

**Examples:**
- **Ticketmaster:** 10-minute lock on seat selection, status-based workflow (available → reserved → sold)
- **Uber:** Lock on driver-rider pair during matching process, TTL prevents stuck matches
- **Rate Limiter:** Redis atomic operations for token bucket

**Trade-offs:**
- ✅ Prevents race conditions and double-booking
- ✅ Auto-expiration prevents deadlocks
- ✅ Simple to implement with Redis
- ❌ Network latency for lock acquisition
- ❌ Clock skew issues in distributed systems
- ❌ Redis becomes single point of failure (use Redis Sentinel/Cluster)

---

### Pattern: Optimistic Locking with Version Fields

**Problem:** Reduce lock contention for resources that are rarely contested.

**Solution:**
- Add version/timestamp field to records
- Read resource with version number
- Update only if version matches (no changes since read)
- Retry on version mismatch

**Implementation Details:**
```sql
-- Read with version
SELECT * FROM inventory WHERE item_id = 123 AND version = 5;

-- Update with version check
UPDATE inventory 
SET quantity = quantity - 1, version = version + 1
WHERE item_id = 123 AND version = 5;

-- If 0 rows updated, version conflict occurred
```

**Examples:**
- **GoPuff:** Inventory updates with version checking
- **E-commerce systems:** Shopping cart checkout with stock version validation

**Trade-offs:**
- ✅ Better performance than pessimistic locking
- ✅ No deadlocks
- ✅ Works well when conflicts are rare
- ❌ Retry logic needed for conflicts
- ❌ User experience degradation on high contention
- ❌ Not suitable for high-contention scenarios

---

#### Deep Dive: Optimistic Locking in Distributed Systems

**Does it work in distributed systems?** Yes, but with important caveats depending on your architecture.

**✅ Works Well: Single Database with Distributed App Servers**

When you have multiple application servers but a single database, optimistic locking works reliably because the **database provides atomicity**:

```sql
-- App Server 1 reads:
SELECT * FROM inventory WHERE item_id = 123 AND version = 5;
-- quantity = 100, version = 5

-- App Server 2 reads (simultaneously):
SELECT * FROM inventory WHERE item_id = 123 AND version = 5;
-- quantity = 100, version = 5

-- App Server 1 updates first:
UPDATE inventory 
SET quantity = 99, version = 6
WHERE item_id = 123 AND version = 5;
-- Success! 1 row updated

-- App Server 2 tries to update:
UPDATE inventory 
SET quantity = 99, version = 6
WHERE item_id = 123 AND version = 5;
-- Fails! 0 rows updated (version is now 6)
-- App Server 2 retries with latest version
```

The version check and update happen **atomically** in a single SQL statement, preventing race conditions.

**⚠️ Challenges: Distributed Databases (Eventual Consistency)**

With eventually consistent databases (Cassandra, DynamoDB default mode), optimistic locking can fail:

```python
# Problem: Replication lag causes conflicts

# Time T0: Item has version=5 on all replicas
# Time T1: Client 1 reads from Replica A (version=5)
# Time T2: Client 2 reads from Replica B (version=5)
# Time T3: Client 1 updates Replica C (version=6) - Success!
# Time T4: Client 2 updates Replica D (version=6) - Success! (hasn't seen T3's update)
# Result: CONFLICT! Both succeeded due to replication lag
```

**Solutions for Distributed Databases:**
1. **Use strong consistency reads** (DynamoDB `ConsistentRead=true`)
2. **Use conditional writes** with version checking
3. **Accept conflicts** and implement conflict resolution (last-write-wins, custom merge)

**❌ More Complex: Multiple Databases (Microservices)**

Across microservices with separate databases, optimistic locking cannot provide atomicity:

```java
// Service A's Database: { order_id: 1, status: "pending", version: 3 }
// Service B's Database: { inventory_id: 5, quantity: 10, version: 2 }

public class OrderService {
    private final OrderServiceClient orderService;
    private final InventoryServiceClient inventoryService;
    
    public void placeOrder(Long orderId, Long itemId) {
        // Read from Service A's database
        Order order = orderService.getOrder(orderId);              // version: 3
        
        // Read from Service B's database
        Inventory inventory = inventoryService.getInventory(itemId); // version: 2
        
        try {
            // Update order in Service A's database
            orderService.updateOrder(orderId, OrderUpdateRequest.builder()
                .status("confirmed")
                .version(order.getVersion() + 1)
                .build());
            // Success!
            
            // Update inventory in Service B's database (network failure here!)
            inventoryService.updateInventory(itemId, InventoryUpdateRequest.builder()
                .quantity(inventory.getQuantity() - 1)
                .version(inventory.getVersion() + 1)
                .build());
            // Fails or times out!
            
        } catch (Exception e) {
            // Result: Inconsistent state (order confirmed but inventory not decremented)
            // No way to rollback Service A since it's a different database
            throw new InconsistentStateException("Order placed but inventory not updated", e);
        }
    }
}
```

For microservices, use **Saga pattern** or **distributed locks** instead.

**Best Practices for Distributed Systems:**

**1. Database-Level Conditional Writes**

PostgreSQL with row-level locking:
```sql
BEGIN TRANSACTION;

SELECT * FROM orders 
WHERE order_id = 123 
FOR UPDATE; -- Prevents concurrent modifications

-- Check version in application
IF fetched_version != expected_version THEN
  ROLLBACK;
  RETURN 'Version conflict';
END IF;

UPDATE orders
SET status = 'confirmed', 
    version = version + 1,
    updated_at = NOW()
WHERE order_id = 123 
  AND version = expected_version;

COMMIT;
```

**2. DynamoDB Conditional Updates**

```javascript
const params = {
  TableName: 'Inventory',
  Key: { itemId: '123' },
  UpdateExpression: 'SET quantity = quantity - :dec, version = version + :inc',
  ConditionExpression: 'version = :expected_version AND quantity >= :dec',
  ExpressionAttributeValues: {
    ':expected_version': 5,
    ':dec': 1,
    ':inc': 1
  }
};

try {
  await dynamodb.update(params);
  console.log('Update successful');
} catch (error) {
  if (error.code === 'ConditionalCheckFailedException') {
    // Version conflict - retry with latest version
    await retryWithLatestVersion();
  }
}
```

**3. Retry Strategy with Exponential Backoff**

```typescript
async function updateWithRetry<T>(
  fetchData: () => Promise<{ data: T; version: number }>,
  applyUpdate: (data: T, version: number) => Promise<boolean>,
  maxRetries: number = 3
): Promise<T> {
  for (let attempt = 0; attempt < maxRetries; attempt++) {
    // Read current version
    const { data, version } = await fetchData();
    
    // Try update with version check
    const success = await applyUpdate(data, version);
    
    if (success) {
      return data;
    }
    
    // Version conflict - exponential backoff before retry
    await sleep(Math.pow(2, attempt) * 100);
  }
  
  throw new Error('Max retries exceeded - high contention detected');
}
```

**When to Use vs Avoid:**

| Scenario | Use Optimistic Locking? | Alternative |
|----------|------------------------|-------------|
| **Single DB, low contention** | ✅ Ideal use case | - |
| **Single DB, high contention** | ❌ Too many retries | Pessimistic locks (Redis) |
| **Distributed DB, strong consistency** | ✅ Use conditional updates | - |
| **Distributed DB, eventual consistency** | ⚠️ Need conflict resolution | Vector clocks, CRDTs |
| **Microservices (multiple DBs)** | ❌ Can't ensure atomicity | Saga pattern, 2PC |
| **Critical operations (payments)** | ❌ Must succeed first time | Distributed locks |

**Hybrid Approach:**

Many systems combine both approaches based on contention levels:

```python
def updateInventory(item_id, quantity_change):
    # Check contention level for this item
    contention_score = get_contention_metric(item_id)
    
    if contention_score > HIGH_CONTENTION_THRESHOLD:
        # High contention: Use pessimistic lock
        with redis_lock(f"inventory:{item_id}", timeout=10):
            return update_with_lock(item_id, quantity_change)
    else:
        # Low contention: Use optimistic lock (faster)
        return update_with_version(item_id, quantity_change, max_retries=3)
```

**Key Takeaway:** Optimistic locking works excellently in distributed **applications** with a centralized database, but requires careful design in truly distributed **databases** or across microservices. The version check and update must be atomic at the storage layer for it to be reliable.

---

### Pattern: ACID Transactions for Atomicity

**Problem:** Ensure multiple operations succeed or fail together (e.g., check inventory + create order).

**Solution:**
- Use ACID-compliant database (PostgreSQL, MySQL)
- Wrap related operations in single transaction
- Database ensures atomicity, consistency, isolation, durability

**Implementation Details:**
```sql
BEGIN TRANSACTION;

-- Check inventory
SELECT quantity FROM inventory WHERE item_id = 123 FOR UPDATE;

-- Deduct inventory if available
UPDATE inventory SET quantity = quantity - 1 WHERE item_id = 123;

-- Create order record
INSERT INTO orders (user_id, item_id, ...) VALUES (...);

COMMIT; -- Or ROLLBACK on error
```

**Examples:**
- **GoPuff:** Single Postgres transaction for inventory check + order creation
- **Ticketmaster:** Transaction for ticket status update + booking record creation
- **Banking systems:** Transfer between accounts

**Trade-offs:**
- ✅ Strong consistency guarantees
- ✅ Simpler than distributed transactions
- ✅ Automatic rollback on failure
- ❌ Locks held during transaction (reduced throughput)
- ❌ Limited to single database
- ❌ Long transactions can cause contention

---

### Pattern: Status-Based Workflow

**Problem:** Track resource lifecycle through multiple states without holding locks continuously.

**Solution:**
- Model resource states explicitly (available → reserved → sold → expired)
- Use status field + timestamp for state transitions
- Background jobs clean up expired states

**Implementation Details:**
```
States: AVAILABLE → RESERVED → SOLD
        ↓ (TTL expired)
        AVAILABLE (cleanup job)

Ticket record:
{
  id, seat_number, event_id,
  status: "RESERVED",
  reserved_at: "2025-11-27T10:00:00Z",
  reserved_until: "2025-11-27T10:10:00Z",
  booking_id: "abc123"
}

Cleanup job (runs every minute):
UPDATE tickets 
SET status = 'AVAILABLE', booking_id = NULL
WHERE status = 'RESERVED' AND reserved_until < NOW();
```

**Examples:**
- **Ticketmaster:** AVAILABLE → RESERVED (10 min) → SOLD workflow
- **Payment processors:** PENDING → AUTHORIZED → CAPTURED → SETTLED
- **Reservation systems:** Hotels, restaurants, appointments

**Trade-offs:**
- ✅ No long-held locks
- ✅ Clear state machine
- ✅ Easy to query by status
- ❌ Cleanup job complexity
- ❌ Small window for race conditions between status check and update
- ❌ Requires periodic background jobs

---

### Pattern: Token Bucket for Rate Limiting

**Problem:** Limit API usage per user/IP without distributed coordination overhead.

**Solution:**
- Each user has "bucket" with fixed capacity (tokens)
- Tokens refill at constant rate
- Request consumes token; rejected if bucket empty
- Use Redis for distributed counter

**Implementation Details:**
```
Redis data structure:
Key: "rate_limit:{user_id}"
Value: {
  tokens: 10,
  last_refill: timestamp,
  capacity: 100,
  refill_rate: 10/second
}

Algorithm:
1. Calculate tokens to add: (now - last_refill) * refill_rate
2. current_tokens = min(tokens + new_tokens, capacity)
3. If current_tokens >= 1:
   - Decrement token
   - Allow request
4. Else: Reject (HTTP 429)

Use Redis EVAL with Lua script for atomicity
```

**Examples:**
- **Distributed Rate Limiter:** Token bucket with Redis atomic operations
- **API Gateway:** Per-user, per-IP, per-endpoint limits
- **Ticketmaster:** Rate limit ticket searches to prevent scraping

**Trade-offs:**
- ✅ Allows burst traffic within limits
- ✅ Fair resource allocation
- ✅ Easy to implement with Redis
- ❌ Hot key problem for popular users
- ❌ Redis latency on every request
- ❌ Need strategy for Redis failure (fail-open vs fail-closed)

---

## Scaling Reads

**When to Use:** Read-heavy workloads where reads vastly outnumber writes (100:1 or higher ratio).

### Pattern: Multi-Layer Caching

**Problem:** Database becomes bottleneck under high read load.

**Solution:**
- Layer 1: In-memory cache (Redis, Memcached) for hot data
- Layer 2: CDN for static assets and geo-distribution
- Layer 3: Application-level cache for computed results
- Layer 4: Database read replicas

**Implementation Details:**
```
Request flow:
1. Check browser cache (client-side)
2. Check CDN edge cache (if applicable)
3. Check Redis cache (TTL: 5-60 min)
4. Check read replica database
5. If miss, query primary DB and populate caches

Cache invalidation:
- Write-through: Update cache on every write
- Write-around: Invalidate cache on write
- TTL-based: Let cache expire naturally (eventual consistency)
```

**Examples:**
- **Bit.ly:** Redis cache for popular short URLs, CDN for redirect responses
- **FB News Feed:** Multi-layer cache for feed posts, profile data
- **Ticketmaster:** Cache event details, venue info (changes infrequently)
- **GoPuff:** Cache inventory availability with short TTL (30-60 sec)
- **YouTube:** CDN for video segments, Redis for metadata

**Cache Strategies by Use Case:**
- **Immutable data** (video files): Long TTL (days), CDN
- **Frequently changing** (inventory): Short TTL (seconds-minutes), Redis
- **User-specific** (feeds): User-keyed cache, invalidate on new content
- **Computed results** (analytics): Pre-compute, cache for hours

**Trade-offs:**
- ✅ Massive reduction in DB load (10-100x)
- ✅ Sub-millisecond response times
- ✅ Reduces cost (cache cheaper than DB queries)
- ❌ Cache invalidation complexity
- ❌ Stale data (eventual consistency)
- ❌ Memory cost for cache storage
- ❌ Cold start problem (cache warming needed)

---

### Pattern: Database Read Replicas

**Problem:** Single database instance can't handle read traffic.

**Solution:**
- Replicate data from primary to multiple read replicas
- Route read queries to replicas
- Write queries only to primary
- Asynchronous replication (eventual consistency)

**Implementation Details:**
```
Architecture:
Primary DB (writes only)
  ↓ (async replication)
Read Replica 1 ───┐
Read Replica 2 ───┼── Load Balancer ← Read queries
Read Replica 3 ───┘

Replication lag: typically 0-5 seconds

Handle replication lag:
1. Read from primary for consistency-critical reads
2. Add "read-your-writes" consistency for user's own data
3. Show "last updated X seconds ago" for acceptable lag
```

**Examples:**
- **GoPuff:** Postgres replicas for availability queries, primary for orders
- **Ticketmaster:** Replicas for event browsing, primary for bookings
- **E-commerce:** Replicas for product catalog, primary for checkout

**Trade-offs:**
- ✅ Horizontal scaling for reads
- ✅ Easy to add more replicas
- ✅ Automatic failover if primary dies
- ❌ Replication lag (eventual consistency)
- ❌ Storage cost multiplied by replica count
- ❌ Complexity in routing (read vs write)

---

### Pattern: Pre-computation & Materialized Views

**Problem:** Complex queries too slow to run in real-time.

**Solution:**
- Pre-compute expensive aggregations/joins
- Store results in fast-access table or cache
- Update asynchronously via background jobs or triggers

**Implementation Details:**
```
Example: User feed generation

Naive approach (slow):
- On request: Query posts from all friends
- Sort by timestamp
- Join with user profiles, likes, comments
- Return top 20 posts

Pre-computed approach (fast):
- Background job: Generate feed for each user every 5 min
- Store in "user_feeds" table or Redis
- On request: SELECT * FROM user_feeds WHERE user_id = X LIMIT 20

Update strategies:
1. Periodic regeneration (every N minutes)
2. Event-driven (new post triggers update)
3. Hybrid (regenerate on first access if stale)
```

**Examples:**
- **FB News Feed:** Pre-compute feed for each user, fan-out on write
- **YouTube Top K:** Pre-aggregate view counts per video per window
- **Analytics dashboards:** Pre-compute daily/weekly aggregations
- **Leaderboards:** Maintain sorted sets in Redis, update on score change

**Trade-offs:**
- ✅ Extremely fast reads (pre-computed)
- ✅ Predictable query performance
- ✅ Reduces real-time computation load
- ❌ Stale data (bounded by update frequency)
- ❌ Storage overhead for materialized data
- ❌ Complex update logic
- ❌ Fan-out cost on writes (e.g., celebrity posts)

---

### Pattern: Pagination & Cursor-based Fetching

**Problem:** Returning large result sets causes memory issues and slow responses.

**Solution:**
- Return data in chunks (pages) with continuation token
- Cursor-based pagination for consistent results
- Offset-based pagination for simpler use cases

**Implementation Details:**
```
Offset-based (simple but inconsistent):
GET /posts?limit=20&offset=40
- Pros: Easy to implement, jump to arbitrary page
- Cons: Results change if data inserted/deleted, slow for large offsets

Cursor-based (recommended for large datasets):
GET /posts?limit=20&cursor=abc123xyz
Response: {
  data: [...],
  next_cursor: "def456uvw"
}

Implementation:
- Cursor encodes position (e.g., last seen ID + timestamp)
- Query: WHERE id > last_id ORDER BY id LIMIT 20
- Client uses next_cursor for next page

Hybrid approach (Tinder):
- Cursor for primary sort key (e.g., seen profiles)
- Bloom filter to avoid showing same profile twice
```

**Examples:**
- **Tinder:** Cursor pagination for profile feed, bloom filter for deduplication
- **FB Live Comments:** Cursor-based for real-time comment stream
- **Search engines:** Offset for first few pages, cursor for deep pagination
- **Ticketmaster:** Paginate search results for events

**Trade-offs:**
- ✅ Constant memory usage regardless of dataset size
- ✅ Cursor-based provides consistent results
- ✅ Better UX (progressive loading)
- ❌ Can't jump to arbitrary page with cursor
- ❌ Complexity in handling sort key changes
- ❌ Cursor may become invalid if data deleted

---

### Pattern: Query Result Caching

**Problem:** Same expensive queries run repeatedly by different users.

**Solution:**
- Cache query results keyed by query parameters
- Share cache across all users for identical queries
- Use cache for read-heavy, infrequently changing data

**Implementation Details:**
```
Cache key generation:
key = hash(query_string + params)
Example: "search:events:keyword=concert:location=NYC:date=2025-12"

Lookup flow:
1. Generate cache key from query params
2. Check Redis: GET cache_key
3. If hit: Return cached result
4. If miss: Execute query, store in Redis with TTL
5. Return result

Invalidation strategies:
- TTL-based: Set appropriate expiration (5 min - 1 hour)
- Event-driven: Invalidate on data change (DELETE cache_key)
- Pattern-based: Invalidate all keys matching pattern
```

**Examples:**
- **Ticketmaster:** Cache search results for popular queries (e.g., "concerts in NYC")
- **E-commerce:** Cache category browsing, filter combinations
- **News sites:** Cache article lists by category/tag
- **Bit.ly:** Cache URL lookups by short code (very high hit rate)

**Trade-offs:**
- ✅ Dramatic reduction in duplicate queries
- ✅ Benefits all users (shared cache)
- ✅ Easy to implement with Redis
- ❌ Cache stampede risk (many requests on cache miss)
- ❌ Invalidation complexity for parameterized queries
- ❌ Memory usage for low-value queries

---

## Scaling Writes

**When to Use:** Write-heavy workloads, high write throughput requirements, or hotspot issues.

### Pattern: Database Sharding

**Problem:** Single database instance can't handle write volume or data size.

**Solution:**
- Partition data across multiple database instances (shards)
- Each shard handles subset of data
- Shard key determines which shard owns the data
- Queries routed to appropriate shard(s)

**Implementation Details:**
```
Sharding strategies:

1. Hash-based sharding:
   shard_id = hash(user_id) % num_shards
   - Even distribution
   - Can't easily add shards (requires rehashing)

2. Range-based sharding:
   Shard 1: user_id 1-1M
   Shard 2: user_id 1M-2M
   - Easy to add shards
   - Risk of hot shards (uneven distribution)

3. Geographic sharding:
   Shard by region (US, EU, APAC)
   - Low latency for users
   - Uneven distribution by region

4. Entity-based sharding:
   Shard by entity type (users, posts, comments)
   - Logical separation
   - Some shards may grow faster than others
```

**Examples:**
- **GoPuff:** Partition inventory by region/distribution center
- **YouTube:** Shard videos by upload date or video_id
- **Social networks:** Shard users by user_id, shard posts by post_id
- **WhatsApp:** Shard messages by chat_id (keeps conversation together)

**Trade-offs:**
- ✅ Linear scaling of writes and storage
- ✅ Isolates failure (one shard down doesn't affect others)
- ✅ Can optimize per-shard (different instance sizes)
- ❌ Complex queries across shards (scatter-gather)
- ❌ Rebalancing difficulty when adding shards
- ❌ Hot shard problem (celebrity users, viral content)
- ❌ Application must be shard-aware

---

### Pattern: Write Batching & Buffering

**Problem:** High frequency of small writes overwhelms database.

**Solution:**
- Buffer writes in memory or message queue
- Flush to database in batches periodically
- Trade latency for throughput

**Implementation Details:**
```
Architecture:
Client → API → Queue (Kafka/SQS) → Batch Writer → Database

Batch writer logic:
1. Accumulate writes in memory buffer
2. Flush when:
   - Buffer size reaches threshold (e.g., 1000 records)
   - Time threshold reached (e.g., 5 seconds)
   - Explicit flush triggered
3. Write batch to DB in single transaction

Example: Analytics events
{
  user_id: 123,
  event_type: "click",
  timestamp: "...",
  metadata: {...}
}

Buffer 1000 events → Bulk INSERT → Reset buffer
```

**Examples:**
- **YouTube Top K:** Batch view count updates, flush every 10 seconds
- **Ad Click Aggregator:** Buffer clicks, batch insert to analytics DB
- **Web Crawler:** Batch URL status updates
- **FB Post Search:** Batch index updates to Elasticsearch

**Trade-offs:**
- ✅ Much higher write throughput (10-100x)
- ✅ Reduces DB connection overhead
- ✅ Better for bulk operations (one transaction vs many)
- ❌ Delayed data visibility (eventual consistency)
- ❌ Risk of data loss if buffer crashes (need WAL/persistence)
- ❌ Memory pressure from buffering
- ❌ Complex error handling (partial batch failures)

---

### Pattern: Stream Processing

**Problem:** Process continuous stream of events in real-time at scale.

**Solution:**
- Use stream processing framework (Kafka, Flink, Kinesis)
- Parallel processing of partitioned streams
- Stateful operations for aggregations, windowing

**Implementation Details:**
```
Architecture:
Event Source → Kafka/Kinesis → Stream Processor (Flink) → Output Sink

Flink operators:
1. Map/Filter: Transform individual events
2. KeyBy: Partition stream by key (for parallel processing)
3. Window: Aggregate over time windows
   - Tumbling: Fixed, non-overlapping (every 1 min)
   - Sliding: Overlapping (last 5 min, updated every 1 min)
   - Session: Dynamic based on event gaps
4. Sink: Write results to DB, cache, another stream

Example: Real-time top videos
Source: video_view events
→ KeyBy(video_id)
→ Window(tumbling, 1 minute)
→ Count()
→ TopK(100)
→ Sink(Redis sorted set)
```

**Examples:**
- **YouTube Top K:** Flink for real-time view count aggregation
- **Ad Click Aggregator:** Flink for click/impression counting by ad_id
- **Uber:** Stream processing for real-time location updates
- **Rate Limiter:** Sliding window counters for rate calculation

**Trade-offs:**
- ✅ Real-time processing (seconds latency)
- ✅ Handles massive throughput (millions events/sec)
- ✅ Built-in fault tolerance (checkpointing)
- ✅ Exactly-once semantics available
- ❌ Operational complexity (managing Flink cluster)
- ❌ Debugging difficulty (distributed state)
- ❌ Learning curve for stream programming model
- ❌ Cost of running processing cluster

---

### Pattern: Asynchronous Processing with Message Queues

**Problem:** Long-running writes block API responses, reducing throughput.

**Solution:**
- Accept write request, return immediately
- Queue work for background processing
- Client polls for completion or receives callback

**Implementation Details:**
```
Synchronous (slow):
Client → API → [Process for 10 seconds] → Response

Asynchronous (fast):
Client → API → Queue → Immediate Response (job_id)
              ↓
         Worker processes in background
              ↓
         Update job status
              ↓
Client polls /jobs/{job_id} or receives webhook

Technologies:
- RabbitMQ, Kafka, AWS SQS, Redis Pub/Sub
- Worker pools (Celery, Sidekiq, Bull)
- Job status tracking (Redis, DynamoDB)
```

**Examples:**
- **YouTube:** Async video transcoding after upload
- **Dropbox:** Async virus scanning, thumbnail generation
- **LeetCode:** Async code execution in sandbox
- **FB News Feed:** Async fan-out on write to followers' feeds
- **Web Crawler:** Async URL fetching and processing

**Trade-offs:**
- ✅ Fast API response times
- ✅ Scales workers independently
- ✅ Natural rate limiting (queue as buffer)
- ✅ Retry logic for failures
- ❌ Eventual consistency (delayed processing)
- ❌ Complex state management
- ❌ Need job status tracking
- ❌ Webhook/polling complexity for clients

---

### Pattern: Write-Ahead Log (WAL) for Durability

**Problem:** Ensure no data loss even if system crashes during write.

**Solution:**
- Append writes to sequential log file before applying
- Log is crash-safe (fsync to disk)
- Replay log on recovery to restore state
- Compact log periodically

**Implementation Details:**
```
Write flow:
1. Append operation to WAL (sequential write)
2. fsync() to ensure on disk
3. Acknowledge write to client
4. Apply to in-memory data structure (async)
5. Periodically checkpoint and truncate WAL

WAL entry format:
[timestamp][operation][key][value][checksum]

Recovery:
1. Read WAL from last checkpoint
2. Replay operations in order
3. Rebuild in-memory state
4. Resume normal operations

Used internally by databases (PostgreSQL, Cassandra, Kafka)
```

**Examples:**
- **Databases:** PostgreSQL WAL, MySQL binlog, Cassandra commitlog
- **Message queues:** Kafka log-based storage
- **Distributed systems:** Raft/Paxos consensus logs
- **Redis:** AOF (Append-Only File) persistence

**Trade-offs:**
- ✅ Durability guarantees (no data loss)
- ✅ Fast writes (sequential, append-only)
- ✅ Point-in-time recovery
- ✅ Replication possible (ship log to replicas)
- ❌ Disk space for logs
- ❌ Periodic compaction needed
- ❌ Recovery time proportional to log size
- ❌ fsync() latency on writes

---

## Handling Large Blobs

**When to Use:** Storing/transferring files >10MB (videos, images, documents, backups).

### Pattern: Presigned URLs for Direct Upload/Download

**Problem:** Uploading/downloading large files through application servers is inefficient.

**Solution:**
- Client requests presigned URL from API server
- API server generates time-limited signed URL to blob storage
- Client uploads/downloads directly to/from S3/blob storage
- Bypasses application server entirely

**Implementation Details:**
```
Upload flow:
1. Client: POST /files/presigned-upload {filename, size, type}
2. Server: 
   - Generate unique file_id
   - Create presigned PUT URL (S3.generatePresignedUrl)
   - Store metadata in DB (file_id, filename, status=uploading)
   - Return: {file_id, presigned_url, expires_in: 3600}
3. Client: PUT presigned_url [file bytes]
4. S3: Direct upload (no app server involved)
5. Client: POST /files/{file_id}/complete
6. Server: Update status=uploaded

Download flow:
1. Client: GET /files/{file_id}/download
2. Server: 
   - Check permissions (ACL)
   - Generate presigned GET URL (S3.generatePresignedUrl)
   - Return: {presigned_url, expires_in: 300}
3. Client: GET presigned_url
4. S3: Direct download

Presigned URL includes:
- Signature (HMAC of URL + timestamp + secret key)
- Expiration timestamp
- Permissions (read/write)
```

**Examples:**
- **Dropbox:** Presigned URLs for file upload/download
- **YouTube:** Presigned URLs for video upload
- **WhatsApp:** Presigned URLs for media attachments
- **Profile images:** Presigned URLs for avatar uploads

**Trade-offs:**
- ✅ No bandwidth through app servers
- ✅ Scales to unlimited file sizes
- ✅ Fast (direct to CDN/storage)
- ✅ Secure (time-limited, signed)
- ❌ Complex client logic (retry, multipart)
- ❌ Harder to enforce upload limits
- ❌ Two-step process (get URL, then upload)

---

### Pattern: Chunking & Multipart Upload

**Problem:** Large files exceed HTTP request size limits, prone to failures, no progress tracking.

**Solution:**
- Split file into chunks (5-10MB each)
- Upload chunks in parallel or sequentially
- Resume from failed chunk on interruption
- Show progress to user

**Implementation Details:**
```
Client-side chunking:
1. Calculate file fingerprint (SHA-256 hash)
2. Split file into 5MB chunks
3. Calculate fingerprint for each chunk

Upload process:
1. POST /files/multipart-init {file_hash, size, chunk_count}
   Response: {upload_id, presigned_urls: [...]}

2. For each chunk:
   PUT presigned_url[i] [chunk bytes]
   Response: {etag: "..."}
   PATCH /files/{upload_id}/chunks/{i} {etag, status: "uploaded"}

3. POST /files/{upload_id}/complete
   Server: Verify all chunks, combine into single object

Resume on failure:
- GET /files/{upload_id}/status → {chunks_uploaded: [0,1,2,5]}
- Resume from chunks [3,4,6,...]

S3 Multipart Upload API:
- initiate_multipart_upload()
- upload_part() for each chunk
- complete_multipart_upload() to finalize
```

**Examples:**
- **Dropbox:** Chunked upload for files >10MB, resumable
- **YouTube:** 256KB chunks for video upload, parallel upload
- **Google Drive:** Resumable uploads with session URLs
- **Backup services:** Chunked backup files for reliability

**Trade-offs:**
- ✅ Resumable uploads (no restart from 0)
- ✅ Progress indicator for users
- ✅ Parallel upload (faster on high bandwidth)
- ✅ Works around size limits
- ❌ Complex client implementation
- ❌ Need to track chunk state
- ❌ More API calls (overhead)
- ❌ Incomplete uploads need cleanup

---

### Pattern: Content-Addressed Storage

**Problem:** Duplicate files waste storage, hard to detect data corruption.

**Solution:**
- Use file hash (SHA-256) as unique identifier
- Store file once, reference multiple times
- Detect duplicates automatically (deduplication)
- Verify integrity on read

**Implementation Details:**
```
Storage:
1. Calculate hash: fingerprint = SHA256(file_bytes)
2. Check if exists: SELECT * FROM files WHERE hash = fingerprint
3. If exists: 
   - Don't upload again (dedupe)
   - Create reference: INSERT INTO user_files (user_id, file_id)
4. If not exists:
   - Upload to S3: PUT s3://bucket/{hash}
   - Store metadata: INSERT INTO files (hash, size, mime_type)

File structure:
/blobs/
  /ab/cd/ef/abcdef123456... (hash-based path)

Metadata:
{
  hash: "abcdef123456...",
  size: 1048576,
  mime_type: "image/jpeg",
  upload_date: "2025-11-27",
  ref_count: 3  // How many users reference this file
}

Garbage collection:
- Decrement ref_count on delete
- DELETE from S3 when ref_count = 0
```

**Examples:**
- **Dropbox:** Deduplication across all users (huge savings)
- **Git:** Content-addressed storage for commits, blobs
- **Docker:** Image layers shared across containers
- **Backup systems:** Dedupe for incremental backups

**Trade-offs:**
- ✅ Massive storage savings (10-90% reduction)
- ✅ Instant "upload" if file exists
- ✅ Built-in integrity checking
- ✅ Efficient for common files
- ❌ Hash collision risk (extremely low with SHA-256)
- ❌ Garbage collection complexity
- ❌ Privacy concerns (file existence leakage)
- ❌ Can't easily update files (immutable)

---

### Pattern: Adaptive Bitrate Streaming

**Problem:** Video streaming needs to work on varying network conditions.

**Solution:**
- Encode video at multiple quality levels (bitrates)
- Split into small segments (2-10 seconds each)
- Client dynamically switches quality based on bandwidth
- Use streaming protocols (HLS, DASH)

**Implementation Details:**
```
Video transcoding:
Original video (4K, 50 Mbps)
  → Transcode to multiple qualities:
    - 4K: 15 Mbps
    - 1080p: 8 Mbps
    - 720p: 5 Mbps
    - 480p: 2.5 Mbps
    - 360p: 1 Mbps

Segmentation:
Each quality level split into 4-second segments:
  video_720p_segment_0001.ts
  video_720p_segment_0002.ts
  ...

Manifest file (m3u8 for HLS):
#EXTM3U
#EXT-X-STREAM-INF:BANDWIDTH=1000000,RESOLUTION=640x360
360p/playlist.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=2500000,RESOLUTION=854x480
480p/playlist.m3u8
...

Client behavior:
1. Download manifest
2. Start with mid-quality (e.g., 720p)
3. Monitor download speed
4. If buffering: Switch to lower quality
5. If bandwidth available: Switch to higher quality
6. Seamless quality switching between segments
```

**Examples:**
- **YouTube:** HLS/DASH with 8+ quality levels
- **Netflix:** Adaptive streaming with per-scene quality optimization
- **Twitch:** Live adaptive streaming
- **Zoom:** Adaptive video quality in calls

**Trade-offs:**
- ✅ Works on slow connections
- ✅ Smooth playback without buffering
- ✅ Optimal quality for available bandwidth
- ✅ Progressive download (start playing immediately)
- ❌ Storage cost (multiple copies of each video)
- ❌ Transcoding cost (CPU-intensive)
- ❌ Complexity in client player
- ❌ Quality switches can be jarring

---

### Pattern: CDN for Global Distribution

**Problem:** Users far from origin servers experience high latency for large files.

**Solution:**
- Distribute content to edge servers worldwide
- Route users to nearest edge location
- Cache popular content at edge
- Reduce load on origin servers

**Implementation Details:**
```
Architecture:
Client → Edge (CDN) → Origin (S3/blob storage)

CDN configuration:
- Origin: s3://my-bucket.s3.us-east-1.amazonaws.com
- Edge locations: 200+ cities worldwide
- Cache rules:
  * Images: Cache for 1 year (immutable)
  * Videos: Cache for 1 week
  * Thumbnails: Cache for 1 day
  * API responses: Cache for 5 minutes

Request flow:
1. Client requests: cdn.example.com/videos/abc123.mp4
2. DNS resolves to nearest edge location (Anycast or GeoDNS)
3. Edge server checks local cache
4. If miss: Fetch from origin, cache, return to client
5. If hit: Return from cache (very fast)

Cache invalidation:
- Time-based: TTL expiration
- Version-based: /videos/abc123_v2.mp4 (new URL)
- Manual purge: API call to CDN provider
```

**Examples:**
- **YouTube:** CloudFront/Akamai for video distribution
- **Dropbox:** CloudFront for file downloads
- **Netflix:** Custom CDN (Open Connect) at ISPs
- **Bit.ly:** CDN for redirect responses (cache redirect)

**Trade-offs:**
- ✅ Low latency worldwide (< 50ms to edge)
- ✅ Massive bandwidth (Tbps)
- ✅ Reduces origin load (95%+ cache hit rate)
- ✅ DDoS protection (traffic absorbed at edge)
- ❌ Cost (per-GB transfer fees)
- ❌ Cache invalidation delays
- ❌ Complex routing rules
- ❌ Less control (managed service)

---

## Real-time Updates

**When to Use:** Users need immediate updates without refreshing (chats, live feeds, notifications, location tracking).

### Pattern: WebSockets for Bi-directional Communication

**Problem:** HTTP polling is inefficient for real-time updates; need push from server.

**Solution:**
- Establish persistent TCP connection (WebSocket)
- Both client and server can send messages anytime
- Low latency (<100ms for message delivery)
- Ideal for chat, gaming, live updates

**Implementation Details:**
```
Connection establishment:
1. Client: HTTP Upgrade request
   GET /chat HTTP/1.1
   Upgrade: websocket
   Connection: Upgrade

2. Server: 101 Switching Protocols
   Upgraded to WebSocket

3. Persistent connection maintained

Message flow:
Client ←→ WebSocket Connection ←→ Server
  ↓                                    ↓
Send message                      Broadcast to recipients
Receive message                   Receive message

Scaling:
- Load balancer with sticky sessions (or L7 routing)
- Pub/Sub for routing messages across servers:
  Server1 → Redis Pub/Sub → Server2
  User A (Server1) sends msg → Publish to Redis
  → Server2 receives → Forward to User B

Architecture:
Client A → WS Server 1 ─┐
Client B → WS Server 2 ─┼→ Redis Pub/Sub → Message Queue → DB
Client C → WS Server 3 ─┘
```

**Examples:**
- **WhatsApp:** WebSocket for real-time message delivery
- **Uber:** WebSocket for driver location updates to rider
- **Slack:** WebSocket for real-time chat messages
- **Stock trading apps:** WebSocket for real-time price updates
- **Online games:** WebSocket for player actions

**Trade-offs:**
- ✅ True real-time (sub-second latency)
- ✅ Bi-directional (server push + client send)
- ✅ Efficient (no polling overhead)
- ✅ Low bandwidth (persistent connection)
- ❌ Connection management complexity
- ❌ Scaling challenges (stateful connections)
- ❌ Firewall/proxy issues
- ❌ Mobile battery drain (persistent connection)

---

### Pattern: Server-Sent Events (SSE) for Server Push

**Problem:** Need server push but not client-to-server real-time messages.

**Solution:**
- HTTP-based unidirectional streaming (server → client)
- Simpler than WebSocket for one-way updates
- Auto-reconnection built-in
- Works over standard HTTP (better firewall compatibility)

**Implementation Details:**
```
Connection:
Client establishes HTTP connection:
GET /live-comments HTTP/1.1
Accept: text/event-stream

Server responds with streaming:
HTTP/1.1 200 OK
Content-Type: text/event-stream
Cache-Control: no-cache

data: {"comment": "Hello!", "user": "Alice"}

data: {"comment": "Nice post", "user": "Bob"}

Event format:
event: comment
id: 12345
retry: 10000
data: {"text": "..."}

Client code (JavaScript):
const source = new EventSource('/live-comments');
source.onmessage = (event) => {
  const data = JSON.parse(event.data);
  updateUI(data);
};

Auto-reconnect:
- If connection drops, client auto-reconnects
- Sends Last-Event-ID header to resume from last received
```

**Examples:**
- **FB Live Comments:** SSE for broadcasting new comments to viewers
- **Live sports scores:** SSE for real-time score updates
- **Stock tickers:** SSE for price updates
- **News feeds:** SSE for breaking news alerts
- **Build systems:** SSE for CI/CD progress updates

**Trade-offs:**
- ✅ Simpler than WebSocket (HTTP-based)
- ✅ Built-in reconnection with last event ID
- ✅ Better firewall/proxy compatibility
- ✅ Easy to implement (native browser API)
- ❌ Unidirectional only (server → client)
- ❌ Limited to text-based messages
- ❌ Browser connection limits (6 per domain)
- ❌ Not as low latency as WebSocket

---

### Pattern: Pub/Sub for Event Broadcasting

**Problem:** Route messages from publishers to many subscribers efficiently.

**Solution:**
- Publishers send messages to topics/channels
- Subscribers listen to topics of interest
- Message broker handles routing and delivery
- Decouple producers from consumers

**Implementation Details:**
```
Architecture:
Publishers → Message Broker (Redis/Kafka/SNS) → Subscribers

Redis Pub/Sub:
// Publisher
PUBLISH chat:room:123 "New message from Alice"

// Subscriber
SUBSCRIBE chat:room:123
// Receives: "New message from Alice"

Topic patterns:
- User-specific: user:{user_id}:notifications
- Room-based: chat:room:{room_id}
- Geographic: location:{city}:events
- Hierarchical: news.sports.football (with wildcard news.sports.*)

Scaling with partitioning:
- Partition topics by key (e.g., chat room ID)
- Each partition handled by different server
- Subscribers connect to partition server
- Reduces load per server

Example: WhatsApp message routing
1. User A sends message to chat XYZ
2. Server publishes to topic: chat:XYZ
3. All subscribers of chat:XYZ receive message
4. Includes User B (on different server)
```

**Examples:**
- **WhatsApp:** Pub/Sub for routing messages to chat participants
- **FB Live Comments:** Pub/Sub channel per video, broadcast to viewers
- **Uber:** Pub/Sub for driver location updates to nearby riders
- **Tinder:** Pub/Sub for match notifications
- **Gaming:** Pub/Sub for game state updates to players

**Trade-offs:**
- ✅ Decouples publishers from subscribers
- ✅ Scales to millions of subscribers
- ✅ Flexible routing (topics, patterns, filters)
- ✅ Easy to add new subscribers
- ❌ Message delivery not guaranteed (fire-and-forget)
- ❌ No message persistence (if subscriber offline)
- ❌ Ordering not always guaranteed
- ❌ Potential message duplication

---

### Pattern: Long Polling for Near Real-time

**Problem:** Need push-like behavior but can't use WebSocket/SSE (firewall, simplicity).

**Solution:**
- Client makes HTTP request, server holds open until data available
- Server responds when new data arrives or timeout
- Client immediately makes new request
- Appears like server push, works over HTTP

**Implementation Details:**
```
Flow:
1. Client: GET /notifications?since=12345
2. Server: 
   - Check for new notifications
   - If found: Return immediately with data
   - If not: Hold connection open (30-60 seconds)
   - Return when notification arrives or timeout
3. Client: 
   - Process response
   - Immediately reconnect with new request

Server pseudo-code:
async function handleLongPoll(userId, since) {
  const timeout = 60_000; // 60 seconds
  const start = Date.now();
  
  while (Date.now() - start < timeout) {
    const newData = await checkForUpdates(userId, since);
    if (newData) return newData;
    await sleep(1000); // Check every second
  }
  
  return { timeout: true }; // No updates
}

Compared to short polling:
- Short polling: Request every 5 sec → 720 requests/hour
- Long polling: Hold for 60 sec → 60 requests/hour (12x reduction)
```

**Examples:**
- **Old Facebook:** Long polling for chat before WebSocket
- **Email clients:** Long polling for new messages
- **Notification systems:** Long polling for push notifications
- **Comet applications:** Pre-WebSocket real-time

**Trade-offs:**
- ✅ Works everywhere (standard HTTP)
- ✅ Simpler than WebSocket
- ✅ Reduces requests vs short polling
- ✅ Near real-time (1-2 second latency)
- ❌ More requests than WebSocket/SSE
- ❌ Server resources held during wait
- ❌ Not true real-time
- ❌ Complex error handling

---

### Pattern: Change Data Capture (CDC) for Data Sync

**Problem:** Keep multiple systems in sync when database changes.

**Solution:**
- Capture database changes (inserts, updates, deletes)
- Stream changes to interested consumers
- Consumers update their own data stores
- Eventually consistent across systems

**Implementation Details:**
```
Architecture:
Database → CDC Tool → Stream (Kafka) → Consumers

CDC approaches:
1. Database triggers:
   CREATE TRIGGER user_update
   AFTER UPDATE ON users
   FOR EACH ROW
   BEGIN
     INSERT INTO change_log (table, operation, data, timestamp)
     VALUES ('users', 'UPDATE', NEW, NOW());
   END;

2. Transaction log tailing:
   - PostgreSQL: Logical replication, wal2json
   - MySQL: Binlog streaming
   - MongoDB: Change streams

3. CDC tools:
   - Debezium (Kafka Connect)
   - AWS DMS (Database Migration Service)
   - Maxwell's Daemon (MySQL)

Change event format:
{
  "table": "users",
  "operation": "UPDATE",
  "before": {"id": 123, "name": "Alice", "email": "old@example.com"},
  "after": {"id": 123, "name": "Alice", "email": "new@example.com"},
  "timestamp": "2025-11-27T10:00:00Z"
}

Consumers:
- Update search index (Elasticsearch)
- Update cache (Redis)
- Update analytics DB (ClickHouse)
- Trigger workflows (send email, update recommendations)
```

**Examples:**
- **FB Post Search:** CDC from posts DB to Elasticsearch index
- **E-commerce:** CDC from inventory DB to search, cache, analytics
- **User profiles:** CDC to update caches, recommendation engines
- **Audit logs:** CDC for compliance and tracking

**Trade-offs:**
- ✅ Decouples systems (loose coupling)
- ✅ Real-time or near real-time sync
- ✅ Single source of truth (database)
- ✅ No application code changes needed
- ❌ Eventual consistency (lag between systems)
- ❌ Operational complexity (CDC infrastructure)
- ❌ Schema evolution challenges
- ❌ Potential data ordering issues

---

## Multi-step Processes & Workflows

**When to Use:** Complex business logic with multiple steps, external dependencies, need for reliability and retry.

### Pattern: Durable Execution with Temporal/Cadence

**Problem:** Multi-step workflows with external calls fail mid-way, hard to retry/recover.

**Solution:**
- Use durable execution framework (Temporal, Cadence)
- Workflow state persisted at each step
- Automatic retry on failures
- Guarantees workflow eventually completes

**Implementation Details:**
```
Example: Uber ride workflow

Workflow steps:
1. Match rider with driver (distributed lock)
2. Send notification to driver (push notification API)
3. Wait for driver acceptance (timeout: 30 seconds)
4. If accepted: Update ride status → "en_route"
5. If timeout: Release lock, try next driver
6. Track driver location (periodic updates)
7. Wait for pickup confirmation
8. Wait for drop-off confirmation
9. Process payment (Stripe API)
10. Send receipts (email service)

Without Temporal (fragile):
- Step 5 fails → Entire workflow lost
- Manual retry logic for each step
- Complex state management
- Difficult to debug

With Temporal (robust):
@WorkflowMethod
public void rideWorkflow(RideRequest request) {
  // Step 1: Match driver
  Driver driver = activities.matchDriver(request);
  
  // Step 2-3: Wait for acceptance (with timeout)
  boolean accepted = Workflow.await(
    Duration.ofSeconds(30),
    () -> isDriverAccepted(driver.id)
  );
  
  if (!accepted) {
    // Retry with next driver
    return rideWorkflow(request);
  }
  
  // Step 4: Update status
  activities.updateRideStatus(request.id, "EN_ROUTE");
  
  // Steps 5-10...
  // Each step automatically retried on failure
  // State persisted after each step
}

Temporal guarantees:
- Workflow executes exactly once
- Automatic retry with exponential backoff
- State persisted (workflow can pause/resume)
- Timeouts handled automatically
- Can run for days/months (durable)
```

**Examples:**
- **Uber:** Ride lifecycle, driver matching, payment processing
- **E-commerce:** Order fulfillment (inventory → payment → shipping → delivery)
- **YouTube:** Video processing pipeline (upload → transcode → thumbnail → index)
- **Airbnb:** Booking workflow (reserve → payment → confirmation → reminders)

**Trade-offs:**
- ✅ Automatic retry and error handling
- ✅ Durable (survives crashes, restarts)
- ✅ Simple workflow code (looks sequential)
- ✅ Built-in timeouts, signals, queries
- ❌ Learning curve (new paradigm)
- ❌ Infrastructure overhead (Temporal cluster)
- ❌ Debugging complexity (distributed state)
- ❌ Cost of running workflow engine

---

### Pattern: Saga Pattern for Distributed Transactions

**Problem:** Transaction spans multiple services, need to ensure all-or-nothing semantics.

**Solution:**
- Break transaction into sequence of local transactions
- Each service commits locally and publishes event
- On failure, execute compensating transactions to rollback
- Two approaches: choreography vs orchestration

**Implementation Details:**
```
Example: E-commerce order placement

Services involved:
- Inventory Service
- Payment Service
- Shipping Service
- Notification Service

Choreography (event-driven):
1. Order Service: Create order → Publish OrderCreated event
2. Inventory Service: Reserve items → Publish ItemsReserved event
3. Payment Service: Charge card → Publish PaymentCompleted event
4. Shipping Service: Create shipment → Publish ShipmentScheduled event
5. Notification Service: Send confirmation email

Failure at step 3 (payment fails):
- Payment Service: Publish PaymentFailed event
- Inventory Service: Listen, release reserved items (compensate)
- Order Service: Listen, mark order as failed

Orchestration (centralized):
OrderSaga orchestrator:
try {
  // Step 1
  inventoryService.reserve(items);
  
  // Step 2
  paymentService.charge(amount);
  
  // Step 3
  shippingService.schedule(order);
  
  // Step 4
  notificationService.sendConfirmation(order);
  
} catch (PaymentError e) {
  // Compensate
  inventoryService.releaseReservation(items);
  notificationService.sendFailure(order);
  throw new OrderFailedException();
}

Compensation actions:
- Inventory.reserve() ←→ Inventory.release()
- Payment.charge() ←→ Payment.refund()
- Shipping.schedule() ←→ Shipping.cancel()
```

**Examples:**
- **E-commerce:** Order placement across multiple services
- **Banking:** Money transfer (debit account A, credit account B)
- **Travel booking:** Flight + hotel + car rental reservation
- **Ticketmaster:** Ticket booking + payment + notification

**Trade-offs:**
- ✅ Works across services/databases
- ✅ No distributed locks needed
- ✅ Services remain independent
- ✅ Better availability than 2PC
- ❌ Eventual consistency (not ACID)
- ❌ Compensation logic complexity
- ❌ Partial states visible to users
- ❌ Idempotency required for all operations

---

### Pattern: DAG-based Processing Pipeline

**Problem:** Complex multi-stage processing with dependencies between stages.

**Solution:**
- Model workflow as Directed Acyclic Graph (DAG)
- Each node is processing stage
- Edges represent dependencies
- Stages execute when dependencies complete
- Parallel execution where possible

**Implementation Details:**
```
Example: Video processing pipeline

DAG structure:
                    Upload
                      ↓
                   Validate
                      ↓
            ┌─────────┴─────────┐
            ↓                   ↓
      Extract Audio        Transcode Video
            ↓                   ↓
    Generate Subtitles     Generate Thumbnails
            ↓                   ↓
            └─────────┬─────────┘
                      ↓
                  Update Index
                      ↓
                   Complete

Parallel execution:
- Extract Audio & Transcode Video run in parallel
- Generate Subtitles & Generate Thumbnails run in parallel
- Update Index waits for all upstream tasks

Orchestration tools:
- Apache Airflow (Python DAGs)
- AWS Step Functions (JSON state machines)
- Temporal (workflow code)
- Prefect, Luigi, Dagster

Airflow example:
with DAG('video_pipeline') as dag:
  upload = PythonOperator(task_id='upload', ...)
  validate = PythonOperator(task_id='validate', ...)
  extract_audio = PythonOperator(task_id='extract_audio', ...)
  transcode = PythonOperator(task_id='transcode', ...)
  
  upload >> validate >> [extract_audio, transcode]

Failure handling:
- Retry individual tasks (with backoff)
- Skip downstream tasks if upstream fails
- Alert on repeated failures
- Manual intervention/replay
```

**Examples:**
- **YouTube:** Video upload → validation → transcoding → thumbnail → indexing
- **Web Crawler:** Fetch → Parse → Extract links → Store → Index
- **Data pipelines:** ETL workflows (extract → transform → load)
- **ML pipelines:** Data prep → training → validation → deployment

**Trade-offs:**
- ✅ Clear dependency visualization
- ✅ Parallel execution (faster)
- ✅ Easy to add/remove stages
- ✅ Retry individual stages
- ❌ Orchestration overhead
- ❌ Debugging complex dependencies
- ❌ Potential for deadlocks (if not truly acyclic)
- ❌ Coordination latency

---

### Pattern: Idempotent Operations

**Problem:** Network retries or failures cause operations to execute multiple times.

**Solution:**
- Design operations to be safely retried
- Same input produces same result, no duplicate side effects
- Use idempotency keys to detect duplicates
- Store processed request IDs

**Implementation Details:**
```
Making operations idempotent:

1. Natural idempotency (SET operations):
   - PUT /user/123 {name: "Alice"} → Always results in same state
   - DELETE /user/123 → Deleting twice is same as deleting once

2. Idempotency with keys (POST operations):
   Client:
   POST /payments
   Idempotency-Key: unique-client-generated-uuid
   Body: {amount: 100, currency: "USD"}
   
   Server:
   if (seenBefore(idempotencyKey)) {
     return cachedResponse(idempotencyKey);
   }
   
   result = processPayment(amount, currency);
   cache(idempotencyKey, result);
   return result;

3. Database-level idempotency:
   INSERT INTO payments (id, amount, status)
   VALUES ('unique-id', 100, 'completed')
   ON CONFLICT (id) DO NOTHING;
   
   -- Or with upsert
   INSERT INTO payments (id, amount, status)
   VALUES ('unique-id', 100, 'completed')
   ON CONFLICT (id) DO UPDATE SET updated_at = NOW();

4. Distributed idempotency (Redis):
   SET idempotency:{key} "processing" NX EX 3600
   if success:
     // First time seeing this key
     process()
     SET idempotency:{key} "completed" EX 86400
   else:
     // Already processing or completed
     return cached result or wait

Idempotency key generation:
- Client-generated UUID
- Hash of request body
- Combination: user_id + request_timestamp + nonce
```

**Examples:**
- **Payment processing:** Stripe idempotency keys prevent double charges
- **Message delivery:** WhatsApp message IDs prevent duplicate messages
- **API retries:** All major APIs support idempotency keys
- **Database writes:** Upsert operations are naturally idempotent

**Trade-offs:**
- ✅ Safe to retry operations
- ✅ Prevents duplicate side effects
- ✅ Simpler error handling
- ✅ Critical for reliable systems
- ❌ Need to store idempotency keys
- ❌ Key expiration management
- ❌ Extra database/cache lookups
- ❌ Client must generate/manage keys

---

## Unique ID Generation

**When to Use:** Need globally unique identifiers for distributed systems (short URLs, user IDs, order IDs).

### Pattern: Counter with Base62 Encoding

**Problem:** Generate short, unique IDs efficiently in distributed system.

**Solution:**
- Maintain global counter (Redis, database)
- Increment atomically for each new ID
- Encode counter value in Base62 (0-9, a-z, A-Z)
- Results in short IDs (6-8 characters)

**Implementation Details:**
```
Base62 encoding:
Characters: 0-9 (10) + a-z (26) + A-Z (26) = 62 characters
Counter: 1234567890
Base62: "1ly7vk"

Length vs capacity:
- 6 chars: 62^6 = 56 billion IDs
- 7 chars: 62^7 = 3.5 trillion IDs
- 8 chars: 62^8 = 218 trillion IDs

Implementation:
// Encode
function toBase62(num) {
  const chars = '0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ';
  let result = '';
  while (num > 0) {
    result = chars[num % 62] + result;
    num = Math.floor(num / 62);
  }
  return result || '0';
}

// Get next ID
function generateShortUrl() {
  // Atomic increment in Redis
  const counter = redis.incr('url_counter');
  const shortCode = toBase62(counter);
  return `https://short.ly/${shortCode}`;
}

Scaling with counter batching:
- Each server requests batch of 1000 IDs
- Redis: INCRBY url_counter 1000 → Returns 50001
- Server uses 50001-51000 locally (no more Redis calls)
- Request new batch when exhausted
```

**Examples:**
- **Bit.ly:** Counter + Base62 for short URLs
- **YouTube:** Base64 encoding for video IDs (11 characters)
- **Instagram:** Modified Base62 for photo IDs
- **Twitter:** Snowflake IDs (timestamp + counter + machine ID)

**Trade-offs:**
- ✅ Short IDs (good for URLs, UX)
- ✅ Simple implementation
- ✅ Sequential (can derive creation order)
- ✅ Efficient (just counter increment)
- ❌ Single point of failure (counter service)
- ❌ Sequential nature may reveal business metrics
- ❌ Hard to shard (global counter)
- ❌ Predictable (security concern)

---

### Pattern: UUID (Universally Unique Identifier)

**Problem:** Generate unique IDs without coordination between nodes.

**Solution:**
- Use UUID v4 (random) or v7 (timestamp + random)
- Each node generates IDs independently
- Collision probability extremely low
- No central coordination needed

**Implementation Details:**
```
UUID v4 (random):
Format: 8-4-4-4-12 hex digits
Example: 550e8400-e29b-41d4-a716-446655440000
Bits: 122 bits random
Collision probability: ~10^-18 for 100 trillion IDs

UUID v7 (time-ordered, recommended):
Format: [timestamp][random]
Example: 017F22E2-79B0-7CC3-98C4-DC0C0C07398F
- First 48 bits: Unix timestamp (millisecond precision)
- Remaining bits: Random
- Benefits: Time-ordered, indexable, still unique

Generation:
import { v4 as uuidv4, v7 as uuidv7 } from 'uuid';

const id_v4 = uuidv4(); // Random
const id_v7 = uuidv7(); // Time-ordered

Storage:
- As string: 36 characters (with dashes)
- As binary: 128 bits (16 bytes)
- As Base64: 22 characters (more compact)

Database indexing:
- UUIDv4: Poor index performance (random insertion)
- UUIDv7: Good index performance (sequential timestamps)
```

**Examples:**
- **Databases:** PostgreSQL UUID type, MySQL BINARY(16)
- **Distributed systems:** Cassandra, DynamoDB partition keys
- **Microservices:** Request IDs, correlation IDs
- **File systems:** Unique file identifiers

**Trade-offs:**
- ✅ No coordination needed (fully distributed)
- ✅ Collision probability near zero
- ✅ Works offline (no network required)
- ✅ UUIDv7 is time-sortable
- ❌ Long (36 chars with dashes, 32 without)
- ❌ Not human-friendly
- ❌ UUIDv4 bad for database indexes
- ❌ Larger storage than integer IDs

---

### Pattern: Snowflake ID (Twitter)

**Problem:** Need unique, sortable IDs with high throughput in distributed system.

**Solution:**
- 64-bit integer combining timestamp + machine ID + sequence
- Time-ordered (sortable by creation time)
- Each machine generates independently
- Very high throughput (thousands/sec per machine)

**Implementation Details:**
```
Snowflake format (64 bits):
[1 bit unused][41 bits timestamp][10 bits machine ID][12 bits sequence]

Bit breakdown:
- 1 bit: Unused (sign bit)
- 41 bits: Timestamp (milliseconds since epoch)
  → Supports 69 years from epoch
- 10 bits: Machine/datacenter ID
  → Supports 1024 machines
- 12 bits: Sequence number
  → 4096 IDs per millisecond per machine

Example:
Timestamp: 1732704000000 (41 bits)
Machine ID: 42 (10 bits)
Sequence: 123 (12 bits)
→ Result: 7156733645524123

Generation:
class Snowflake {
  constructor(machineId) {
    this.machineId = machineId;
    this.sequence = 0;
    this.lastTimestamp = 0;
  }
  
  nextId() {
    let timestamp = Date.now();
    
    if (timestamp === this.lastTimestamp) {
      this.sequence = (this.sequence + 1) & 0xFFF; // 12 bits
      if (this.sequence === 0) {
        // Wait for next millisecond
        while (timestamp <= this.lastTimestamp) {
          timestamp = Date.now();
        }
      }
    } else {
      this.sequence = 0;
    }
    
    this.lastTimestamp = timestamp;
    
    return (timestamp << 22) | (this.machineId << 12) | this.sequence;
  }
}

Properties:
- Time-sortable (ORDER BY id = ORDER BY created_at)
- Roughly 4 million IDs per second per machine
- 64-bit integer (smaller than UUID, faster comparison)
```

**Examples:**
- **Twitter:** Tweet IDs, user IDs
- **Discord:** Message IDs, channel IDs
- **Instagram:** Photo IDs (modified Snowflake)
- **Distributed databases:** TiDB, CockroachDB

**Trade-offs:**
- ✅ Time-sortable (useful for pagination)
- ✅ Compact (64-bit integer)
- ✅ High throughput (millions per second)
- ✅ No coordination between machines
- ❌ Clock synchronization required (NTP)
- ❌ Machine ID management (assignment, reuse)
- ❌ Reveals creation time (privacy concern)
- ❌ Limited to 1024 machines (can adjust bit allocation)

---

## Geospatial & Location-based Services

**When to Use:** Location-based queries (nearby restaurants, available drivers, local events).

### Pattern: Geohashing for Proximity Search

**Problem:** Find all points within certain distance of a location efficiently.

**Solution:**
- Encode lat/long into single string (geohash)
- Nearby locations share common prefix
- Query by prefix for approximate proximity
- Refine with distance calculation

**Implementation Details:**
```
Geohash encoding:
Location: (37.7749, -122.4194) // San Francisco
Geohash: "9q8yy" (5 characters, ~5km precision)
Geohash: "9q8yy9r" (7 characters, ~150m precision)

Precision levels:
1 char: ±2500 km
2 char: ±630 km
3 char: ±78 km
4 char: ±20 km
5 char: ±2.4 km (typical for "nearby" queries)
6 char: ±610 m
7 char: ±76 m

Storage and indexing:
CREATE TABLE restaurants (
  id INT,
  name VARCHAR(255),
  lat DECIMAL(9,6),
  lng DECIMAL(9,6),
  geohash CHAR(7),
  INDEX idx_geohash (geohash)
);

Query nearby:
-- User location: 9q8yy9r
SELECT * FROM restaurants
WHERE geohash LIKE '9q8yy%'
AND distance(lat, lng, user_lat, user_lng) < 5000  -- 5km
ORDER BY distance
LIMIT 20;

Adjacent cells:
- Geohash cells are grid-based
- Query needs to check adjacent cells for edge cases
- User at edge of cell needs results from neighbors

Neighbor calculation:
const neighbors = getGeohashNeighbors('9q8yy9r');
// Returns 8 neighboring cells
WHERE geohash IN (center, north, south, east, west, ...)
```

**Examples:**
- **Tinder:** Nearby profiles using geohash
- **Uber:** Find nearby drivers
- **Yelp:** Nearby restaurants, businesses
- **Pokemon Go:** Nearby Pokemon, gyms

**Trade-offs:**
- ✅ Fast proximity queries (index-based)
- ✅ Works with standard databases
- ✅ Prefix-based (easy to shard)
- ✅ Hierarchical (zoom in/out)
- ❌ Approximate (need distance refinement)
- ❌ Edge cases (cell boundaries)
- ❌ Non-uniform cell sizes near poles
- ❌ Need to query multiple cells

---

### Pattern: Redis Geospatial Indexes

**Problem:** Real-time geospatial queries with high performance.

**Solution:**
- Use Redis sorted set with geospatial commands
- Store locations with lat/long
- Query by radius or bounding box
- Sub-millisecond query times

**Implementation Details:**
```
Redis geospatial commands:

GEOADD (add locations):
GEOADD drivers 
  -122.4194 37.7749 "driver:1"
  -122.4084 37.7849 "driver:2"
  -122.4300 37.7650 "driver:3"

GEORADIUS (find within radius):
GEORADIUS drivers -122.4194 37.7749 5 km
  WITHDIST        -- Include distance
  WITHCOORD       -- Include coordinates
  COUNT 10        -- Limit results
  ASC             -- Nearest first

Response:
1) "driver:1" (0.0 km)
2) "driver:2" (1.2 km)
3) "driver:3" (1.5 km)

GEORADIUS_RO (read-only version):
- Same as GEORADIUS but doesn't modify key

GEORADIUSBYMEMBER (radius from existing member):
GEORADIUSBYMEMBER drivers "driver:1" 5 km

Update location:
GEOADD drivers -122.4200 37.7755 "driver:1"  -- Upsert

Remove:
ZREM drivers "driver:1"

Internal representation:
- Uses Sorted Set with geohash as score
- Sorted Set enables range queries
- Geohash provides spatial ordering
```

**Examples:**
- **Uber:** Real-time driver location, nearby rider matching
- **Food delivery:** Find nearby restaurants, available drivers
- **Dating apps:** Find users within X km
- **Real estate:** Properties within area

**Trade-offs:**
- ✅ Very fast queries (<1ms)
- ✅ Easy to use (built-in Redis commands)
- ✅ Handles updates efficiently
- ✅ Radius and bounding box queries
- ❌ In-memory only (RAM cost)
- ❌ Limited to Redis capacity
- ❌ Need persistence/replication strategy
- ❌ Not suitable for historical data

---

### Pattern: S2 Geometry for Precise Coverage

**Problem:** Geohashing has issues with cell boundaries and uneven sizes.

**Solution:**
- Use S2 geometry library (Google)
- Projects sphere onto cube faces
- Hierarchical cells of uniform size
- Better coverage and precision

**Implementation Details:**
```
S2 cell hierarchy:
- Level 0: 6 cells (cube faces)
- Level 1: 24 cells
- Level 2: 96 cells
- ...
- Level 30: ~1 cm² cells

Cell ID encoding:
Location: (37.7749, -122.4194)
S2 Cell (level 15): 0x89c2594c
S2 Cell (level 20): 0x89c2594c3

Region covering:
// Cover a 5km radius circle
const region = S2.RegionCoverer({
  min_level: 10,
  max_level: 15,
  max_cells: 50
});

const cells = region.getCovering(circle(lat, lng, 5000));
// Returns [cell_id_1, cell_id_2, ..., cell_id_n]

Query:
WHERE s2_cell_id IN (cell_id_1, cell_id_2, ..., cell_id_n)

Advantages over geohash:
- Uniform cell sizes (square meters)
- Better boundary handling
- Precise covering of arbitrary shapes
- Used by Google Maps, Uber
```

**Examples:**
- **Uber:** Driver-rider matching with S2 cells
- **Google Maps:** Location indexing
- **Foursquare:** Venue check-ins
- **Rideshare apps:** Service area definitions

**Trade-offs:**
- ✅ More precise than geohash
- ✅ Uniform cell sizes
- ✅ Better for complex shapes
- ✅ Hierarchical (zoom levels)
- ❌ More complex than geohash
- ❌ Requires S2 library
- ❌ Harder to visualize
- ❌ More storage (64-bit cell IDs)

---

## Search & Full-text Indexing

**When to Use:** Text search, keyword matching, fuzzy search, ranked results.

### Pattern: Inverted Index for Text Search

**Problem:** Find documents containing specific keywords efficiently.

**Solution:**
- Build index mapping words → documents containing them
- Tokenize documents into terms
- Store term positions for phrase matching
- Support boolean queries (AND, OR, NOT)

**Implementation Details:**
```
Document corpus:
Doc 1: "The quick brown fox jumps"
Doc 2: "The lazy brown dog sleeps"
Doc 3: "Quick dogs and foxes"

Inverted index:
Term        → Document IDs (with positions)
"the"       → [1:[0], 2:[0]]
"quick"     → [1:[1], 3:[0]]
"brown"     → [1:[2], 2:[2]]
"fox"       → [1:[3]]
"jumps"     → [1:[4]]
"lazy"      → [2:[1]]
"dog"       → [2:[3], 3:[1]]
"sleeps"    → [2:[4]]
"foxes"     → [3:[3]]

Query: "brown dog"
1. Lookup "brown" → [1, 2]
2. Lookup "dog" → [2, 3]
3. Intersect (AND) → [2]
Result: Doc 2

Query: "quick dog"
1. Lookup "quick" → [1, 3]
2. Lookup "dog" → [2, 3]
3. Intersect → [3]
Result: Doc 3

Text processing:
1. Tokenization: "The quick brown" → ["the", "quick", "brown"]
2. Lowercase: ["the", "quick", "brown"]
3. Stop word removal: ["quick", "brown"] (remove "the")
4. Stemming: ["quick", "brown"] → ["quick", "brown"]
5. Index each term

Phrase search ("brown fox"):
- Check if "brown" and "fox" are adjacent in doc
- Use position information: brown@2, fox@3 → Match!

Boolean queries:
- "cat AND dog" → Intersect posting lists
- "cat OR dog" → Union posting lists
- "cat NOT dog" → Difference
```

**Examples:**
- **FB Post Search:** Inverted index for post text search
- **Search engines:** Core data structure for Google, Bing
- **Code search:** GitHub code search
- **Document search:** Confluence, Notion

**Trade-offs:**
- ✅ Fast keyword lookup (O(1) per term)
- ✅ Boolean queries efficient
- ✅ Phrase matching possible
- ✅ Compact (only index unique terms)
- ❌ Index size grows with vocabulary
- ❌ Updates require reindexing
- ❌ Ranking requires additional data (TF-IDF, BM25)
- ❌ No fuzzy matching (typos)

---

### Pattern: Elasticsearch for Full-text Search

**Problem:** Need production-ready search with ranking, fuzzy matching, analytics.

**Solution:**
- Use Elasticsearch (built on Lucene)
- Distributed search cluster
- Full-text search with relevance scoring
- Aggregations for faceted search

**Implementation Details:**
```
Index structure:
POST /posts/_doc/1
{
  "title": "System Design Patterns",
  "body": "Learn about caching, sharding, and load balancing",
  "author": "Alice",
  "tags": ["system-design", "architecture"],
  "created_at": "2025-11-27T10:00:00Z"
}

Search query:
GET /posts/_search
{
  "query": {
    "multi_match": {
      "query": "caching patterns",
      "fields": ["title^2", "body"],  // Boost title 2x
      "fuzziness": "AUTO"              // Allow typos
    }
  },
  "highlight": {
    "fields": {"body": {}}
  },
  "from": 0,
  "size": 20
}

Response:
{
  "hits": {
    "total": 42,
    "hits": [
      {
        "_score": 5.2,
        "_source": {...},
        "highlight": {
          "body": ["Learn about <em>caching</em>, sharding"]
        }
      }
    ]
  }
}

Relevance scoring (BM25):
- Term frequency (TF): How often term appears in doc
- Inverse document frequency (IDF): Rarity of term
- Field length normalization: Penalize long docs
- Score = Σ(IDF * TF * field_boost)

Aggregations (faceted search):
GET /posts/_search
{
  "aggs": {
    "by_author": {
      "terms": {"field": "author"}
    },
    "by_date": {
      "date_histogram": {
        "field": "created_at",
        "interval": "month"
      }
    }
  }
}

Sharding for scale:
- Index split into N shards
- Each shard is self-contained Lucene index
- Query scattered to all shards, results merged
- Replicas for availability
```

**Examples:**
- **FB Post Search:** Elasticsearch for full-text search of posts
- **E-commerce:** Product search with filters/facets
- **Ticketmaster:** Event search by name, location, date
- **Log aggregation:** ELK stack (Elasticsearch, Logstash, Kibana)

**Trade-offs:**
- ✅ Production-ready (clustering, replication)
- ✅ Relevance scoring built-in
- ✅ Fuzzy matching for typos
- ✅ Aggregations for analytics
- ❌ Operational complexity (cluster management)
- ❌ Memory intensive
- ❌ Eventually consistent (near real-time, not immediate)
- ❌ Costly for small use cases

---

### Pattern: Prefix/Trie for Autocomplete

**Problem:** Suggest completions as user types (search suggestions, autocomplete).

**Solution:**
- Build trie (prefix tree) of terms
- Each node represents a character
- Traverse trie to find matching prefixes
- Return top-k completions by popularity

**Implementation Details:**
```
Trie structure:
Words: ["cat", "car", "card", "care", "dog"]

      root
     /    \
    c      d
    |      |
    a      o
   / \     |
  t   r    g
      |
      e,d

Search "ca":
1. Traverse: root → c → a
2. Find all words under this prefix: ["cat", "car", "card", "care"]
3. Return top-k by score/popularity

Weighted trie:
- Store weight/score at leaf nodes
- Weight = search frequency, popularity, etc.
- Return top-k by weight

Prefix search:
class TrieNode {
  children: Map<char, TrieNode>;
  isEnd: boolean;
  weight: number;
}

function search(prefix) {
  node = traverse(root, prefix);
  results = collectAllWords(node);
  return results.sort((a, b) => b.weight - a.weight).slice(0, 10);
}

Space optimization:
- Compressed trie (DAWG): Merge common suffixes
- Store only top-k completions at each node
- Prune low-frequency terms

Redis implementation:
- Use Sorted Set for each prefix
- ZADD prefix:ca 100 "cat" 50 "car" 200 "card"
- ZREVRANGE prefix:ca 0 9  // Top 10
```

**Examples:**
- **Google Search:** Autocomplete suggestions
- **E-commerce:** Product name autocomplete
- **IDE:** Code completion
- **Address input:** City, street name suggestions

**Trade-offs:**
- ✅ Very fast prefix lookup (O(k) where k = prefix length)
- ✅ Memory efficient with compression
- ✅ Easy to update (insert/delete)
- ✅ Natural ranking by popularity
- ❌ Space complexity (large vocabulary)
- ❌ Fuzzy matching requires additional logic
- ❌ Multi-word queries more complex
- ❌ Cold start (need popularity data)

---

## Data Structures for Scale

**When to Use:** Approximate counting, membership testing, cardinality estimation at massive scale.

### Pattern: Bloom Filter for Membership Testing

**Problem:** Check if element exists in large set without storing entire set.

**Solution:**
- Probabilistic data structure (space-efficient)
- Can have false positives, never false negatives
- Use k hash functions to set k bits
- Query checks if all k bits are set

**Implementation Details:**
```
Bloom filter:
- Bit array of size m
- k hash functions
- n elements added

Insert "alice":
1. hash1("alice") = 5  → Set bit 5
2. hash2("alice") = 12 → Set bit 12
3. hash3("alice") = 23 → Set bit 23

Query "alice":
1. Check bits 5, 12, 23
2. All set? → Probably in set
3. Any unset? → Definitely NOT in set

False positive rate:
p ≈ (1 - e^(-kn/m))^k
- m = 10,000 bits, k = 3, n = 1000 elements
- p ≈ 0.02 (2% false positive rate)

Optimal parameters:
k = (m/n) * ln(2)  // Optimal number of hash functions
m = -n*ln(p) / (ln(2)^2)  // Required bits for target p

Use cases:
- Check if username taken (avoid DB query)
- Detect duplicate URLs in crawler
- Cache miss prediction
- Spell check (word in dictionary)

Redis implementation:
BF.ADD users "alice"
BF.EXISTS users "alice"  → 1 (probably exists)
BF.EXISTS users "bob"    → 0 (definitely doesn't exist)
```

**Examples:**
- **Web Crawler:** Avoid revisiting URLs (Bloom filter of seen URLs)
- **Tinder:** Avoid showing same profile twice (Bloom filter of seen profiles)
- **Databases:** Reduce disk reads (check if key might exist)
- **CDN:** Check if content in cache before querying origin

**Trade-offs:**
- ✅ Extremely space-efficient (bits per element)
- ✅ Fast lookups (O(k) hash operations)
- ✅ Fixed memory (doesn't grow with elements)
- ✅ Good for "probably in set" queries
- ❌ False positives (tunable rate)
- ❌ Can't delete elements (counting Bloom filter variant helps)
- ❌ Can't list all elements
- ❌ Needs careful sizing upfront

---

### Pattern: Count-Min Sketch for Frequency Estimation

**Problem:** Estimate frequency of items in stream without storing all counts.

**Solution:**
- Approximate counting with fixed memory
- Multiple hash functions with counters
- Query returns minimum of counter values
- Overestimates (never underestimates)

**Implementation Details:**
```
Count-Min Sketch:
- 2D array: depth d, width w
- d hash functions (one per row)
- Each cell is counter

Structure (d=3, w=10):
Row 1: [5][2][8][1][3][9][4][6][2][7]
Row 2: [3][7][4][2][6][1][8][5][3][9]
Row 3: [2][4][6][8][3][5][7][1][9][4]

Update "alice":
1. hash1("alice") % 10 = 3 → Increment row1[3]
2. hash2("alice") % 10 = 5 → Increment row2[5]
3. hash3("alice") % 10 = 7 → Increment row3[7]

Query "alice":
1. Get row1[3] = 8
2. Get row2[5] = 1
3. Get row3[7] = 7
4. Return min(8, 1, 7) = 1  // Estimated count

Error bounds:
- Overestimate by at most ε*n with probability 1-δ
- Space: O((1/ε) * log(1/δ))
- Example: ε=0.01, δ=0.01 → ~2KB for millions of items

Typical parameters:
- w = ⌈e / ε⌉ (width)
- d = ⌈ln(1/δ)⌉ (depth)
- For ε=0.001, δ=0.01: w≈2718, d≈5
```

**Examples:**
- **YouTube Top K:** Estimate view counts for videos
- **Ad Click Aggregator:** Approximate click counts for ads
- **Network traffic:** Count packet frequencies by source IP
- **Real-time analytics:** Trending hashtags, popular products

**Trade-offs:**
- ✅ Fixed memory (independent of item count)
- ✅ Fast updates and queries (O(d))
- ✅ Mergeable (combine multiple sketches)
- ✅ Good for heavy hitters (top-k)
- ❌ Approximate (overestimates)
- ❌ Can't decrease counts
- ❌ Error increases with total count
- ❌ Need to tune ε and δ

---

### Pattern: HyperLogLog for Cardinality Estimation

**Problem:** Count distinct elements (cardinality) in very large datasets.

**Solution:**
- Probabilistic cardinality estimator
- Uses ~12KB for any cardinality (up to billions)
- Standard error ~0.81% (very accurate)
- Mergeable across multiple instances

**Implementation Details:**
```
HyperLogLog algorithm:
1. Hash each element (64-bit hash)
2. Use first b bits as bucket index (2^b buckets)
3. Count leading zeros in remaining bits
4. Store max leading zeros per bucket
5. Estimate cardinality from bucket maxima

Example (simplified):
Element: "alice"
Hash: 0b0010110...11001 (64 bits)
Bucket (first 14 bits): 0b00101100000000 = 2816
Leading zeros in rest: 3
Update: buckets[2816] = max(buckets[2816], 3)

Cardinality estimate:
E = α * m^2 * (Σ 2^(-M[j]))^(-1)
- m = number of buckets (typically 2^14 = 16384)
- M[j] = max leading zeros in bucket j
- α = constant (0.7213...)

Redis HyperLogLog:
PFADD users "alice" "bob" "charlie"
PFCOUNT users  → 3 (approximately)

PFADD users "alice" "david"
PFCOUNT users  → 4

Merging:
PFADD users1 "alice" "bob"
PFADD users2 "charlie" "david"
PFMERGE users users1 users2
PFCOUNT users  → 4

Memory:
- Standard: 12KB per HLL (16384 buckets * 6 bits)
- Sparse: 100-1000 bytes for small cardinalities
```

**Examples:**
- **Website analytics:** Unique visitors per day
- **Database query optimization:** Estimate distinct values for index selection
- **A/B testing:** Count unique users per variant
- **Social networks:** Estimate reach (unique viewers)

**Trade-offs:**
- ✅ Fixed memory (12KB for any cardinality)
- ✅ Very accurate (0.81% standard error)
- ✅ Mergeable (union of sets)
- ✅ Fast (O(1) add, O(m) count where m=buckets)
- ❌ Approximate (not exact)
- ❌ Can't list elements
- ❌ Can't remove elements
- ❌ Only counts cardinality (no other stats)

---

## Technology Stack Reference

### Databases

**Relational (SQL):**
- **PostgreSQL:** ACID transactions, JSON support, full-text search, geospatial (PostGIS)
  - Use for: Complex queries, consistency-critical, structured data
  - Examples: GoPuff inventory, Ticketmaster bookings, user profiles
  
- **MySQL:** Mature, widely adopted, good replication
  - Use for: Web applications, read-heavy workloads
  - Examples: WordPress, legacy systems

**NoSQL:**
- **DynamoDB:** Key-value, managed AWS, single-digit ms latency
  - Use for: High throughput, flexible schema, session storage
  - Examples: FB News Feed metadata, Dropbox file metadata
  
- **Cassandra:** Wide-column, write-optimized, multi-datacenter
  - Use for: Time-series, high write throughput, always available
  - Examples: WhatsApp message history, IoT data
  
- **MongoDB:** Document store, flexible schema
  - Use for: Rapid development, JSON documents, hierarchical data
  - Examples: Content management, catalogs

**Search:**
- **Elasticsearch:** Full-text search, analytics, log aggregation
  - Use for: Text search, log analysis, faceted search
  - Examples: FB Post Search, Ticketmaster event search, product catalogs
  
- **Algolia:** Managed search, typo-tolerance, geo-search
  - Use for: User-facing search, autocomplete
  - Examples: E-commerce product search

**Analytics/OLAP:**
- **ClickHouse:** Columnar, fast aggregations, SQL
  - Use for: Real-time analytics, dashboards
  - Examples: Ad Click Aggregator, YouTube analytics
  
- **BigQuery:** Serverless, petabyte-scale, SQL
  - Use for: Data warehousing, batch analytics
  - Examples: Business intelligence, data science

**Time-series:**
- **TimescaleDB:** PostgreSQL extension for time-series
  - Use for: Metrics, IoT, financial data
  - Examples: Monitoring systems, sensor data

---

### Caching & In-Memory

**Redis:**
- Data structures: String, Hash, List, Set, Sorted Set, Geospatial
- Use cases:
  - Caching (TTL, LRU eviction)
  - Distributed locks (Redlock)
  - Rate limiting (counters, sliding windows)
  - Pub/Sub (real-time messaging)
  - Leaderboards (sorted sets)
  - Geospatial queries (GEORADIUS)
- Examples: Ticketmaster locks, Bit.ly cache, Uber driver locations

**Memcached:**
- Simple key-value cache
- Use for: Session storage, database query cache
- Faster than Redis for simple get/set, no persistence

**CDN:**
- **CloudFront (AWS):** Global edge network, integrates with S3
- **Cloudflare:** DDoS protection, edge computing (Workers)
- **Akamai:** Enterprise CDN, media streaming
- Use for: Static assets, video streaming, API caching
- Examples: YouTube videos, Dropbox downloads, Bit.ly redirects

---

### Message Queues & Streaming

**Apache Kafka:**
- Distributed event streaming, high throughput (millions msg/sec)
- Features: Topics, partitions, consumer groups, log retention
- Use cases: Event sourcing, stream processing, CDC
- Examples: FB News Feed fan-out, YouTube view events, log aggregation

**AWS SQS:**
- Managed queue, at-least-once delivery
- Features: Standard (best-effort ordering) vs FIFO (exactly-once)
- Use for: Decoupling services, async processing
- Examples: Web Crawler URL queue, background jobs

**RabbitMQ:**
- Message broker, routing flexibility (exchanges, queues)
- Use for: Task queues, RPC, complex routing
- Examples: Microservices communication

**AWS Kinesis:**
- Managed streaming (similar to Kafka)
- Use for: Real-time analytics, log processing
- Examples: Clickstream analysis, IoT data ingestion

---

### Stream Processing

**Apache Flink:**
- Stateful stream processing, exactly-once semantics
- Features: Windowing, event time processing, checkpointing
- Use cases: Real-time aggregations, ETL, fraud detection
- Examples: YouTube Top K, Ad Click Aggregator, real-time analytics

**Kafka Streams:**
- Stream processing library (runs with Kafka)
- Use for: Lightweight stream processing, transformations
- Examples: Real-time data enrichment

**Spark Streaming:**
- Micro-batch processing (not true streaming)
- Use for: Batch + streaming workloads
- Examples: ETL pipelines, ML feature engineering

---

### Object Storage

**AWS S3:**
- Object storage, 99.999999999% durability
- Features: Versioning, lifecycle policies, events, multipart upload
- Use cases: Files, videos, backups, data lakes
- Examples: Dropbox files, YouTube videos, static websites

**Google Cloud Storage:**
- Similar to S3, multi-regional replication
- Use for: Same as S3 (alternative cloud provider)

**Azure Blob Storage:**
- Microsoft's object storage
- Use for: Same as S3 (alternative cloud provider)

---

### Load Balancing

**Layer 4 (TCP/UDP):**
- AWS NLB, HAProxy
- Routes based on IP, port
- Use for: High throughput, low latency, simple routing

**Layer 7 (HTTP/Application):**
- AWS ALB, NGINX, Envoy
- Routes based on URL, headers, cookies
- Features: Path-based routing, sticky sessions, SSL termination
- Use for: Microservices, WebSocket upgrades, A/B testing
- Examples: Ticketmaster (WebSocket routing), API Gateway

---

### Workflow Orchestration

**Temporal:**
- Durable execution, automatic retry, timeouts
- Use for: Multi-step workflows, long-running processes
- Examples: Uber ride workflow, order fulfillment, payment processing

**Apache Airflow:**
- DAG-based workflow, Python
- Use for: Data pipelines, batch jobs, ETL
- Examples: YouTube transcoding, Web Crawler, ML pipelines

**AWS Step Functions:**
- Serverless workflows, JSON state machines
- Use for: AWS-native workflows, Lambda orchestration
- Examples: Order processing, approval workflows

---

### Container Orchestration

**Kubernetes:**
- Container orchestration, auto-scaling, self-healing
- Use for: Microservices, stateful applications, multi-cloud
- Examples: Large-scale deployments, production workloads

**Docker Swarm:**
- Simpler than K8s, good for small/medium deployments
- Use for: Small teams, simpler requirements

**AWS ECS/Fargate:**
- Managed container service
- Use for: AWS-native, serverless containers

---

### Security & Auth

**OAuth 2.0 / OpenID Connect:**
- Industry standard for authorization/authentication
- Use for: Third-party login, API access tokens

**JWT (JSON Web Tokens):**
- Stateless auth tokens
- Use for: API authentication, microservices

**AWS IAM:**
- Identity and access management
- Use for: AWS resource permissions, service-to-service auth

---

## Problem-to-Pattern Mapping

### Ticketmaster
**Patterns Used:**
- Dealing with Contention: Distributed lock with TTL, status-based workflow
- Scaling Reads: Caching event details, read replicas
- Real-time Updates: Virtual waiting queue for popular events
- Search: Elasticsearch for event search, query result caching

**Key Technologies:** PostgreSQL, Redis (locks), Elasticsearch, Stripe (payments)

---

### Dropbox
**Patterns Used:**
- Handling Large Blobs: Presigned URLs, chunking/multipart upload, content-addressed storage
- Scaling Reads: CDN for downloads, CloudFront signed URLs
- Real-time Updates: WebSocket/SSE for file sync notifications
- Unique IDs: Fingerprinting (SHA-256) for deduplication

**Key Technologies:** S3, CloudFront CDN, WebSocket, PostgreSQL/DynamoDB

---

### Bit.ly
**Patterns Used:**
- Unique ID Generation: Counter with Base62 encoding
- Scaling Reads: Multi-layer caching (Redis + CDN), database indexes
- Scaling Writes: Counter batching to reduce Redis load

**Key Technologies:** Redis (counter + cache), PostgreSQL, CDN

---

### GoPuff
**Patterns Used:**
- Dealing with Contention: ACID transactions in Postgres, atomic inventory updates
- Scaling Reads: Redis cache for availability, read replicas
- Geospatial: PostgreSQL distance calculation, nearby service

**Key Technologies:** PostgreSQL, Redis, travel time API integration

---

### FB News Feed
**Patterns Used:**
- Scaling Reads: Pre-computation (fan-out on write), caching user feeds
- Scaling Writes: Async workers for feed generation, sharding by user
- Multi-step Processes: Async fan-out to millions of followers

**Key Technologies:** DynamoDB, Redis, Kafka, async workers

---

### WhatsApp
**Patterns Used:**
- Real-time Updates: WebSocket for bi-directional messaging, pub/sub for routing
- Scaling Writes: Sharding by chat_id, Cassandra for message history
- Multi-step Processes: Multi-device sync, message persistence (30 days)

**Key Technologies:** WebSocket, Cassandra, Redis Pub/Sub, Erlang

---

### FB Live Comments
**Patterns Used:**
- Real-time Updates: SSE for broadcasting comments, pub/sub per video
- Scaling Reads: Layer 7 load balancing by video_id, edge caching
- Pagination: Cursor-based for comment stream

**Key Technologies:** SSE, Redis Pub/Sub, Layer 7 LB, DynamoDB

---

### YouTube Top K
**Patterns Used:**
- Stream Processing: Flink for real-time view count aggregation
- Scaling Writes: Batching view events, tumbling windows
- Data Structures: Count-Min Sketch for approximate counting
- Scaling Reads: Pre-computed top-k stored in Redis sorted set

**Key Technologies:** Apache Flink, Kafka, Redis, OLAP DB (ClickHouse)

---

### Distributed Rate Limiter
**Patterns Used:**
- Dealing with Contention: Token bucket algorithm, Redis atomic operations
- Scaling Writes: Distributed counters, local cache + periodic sync

**Key Technologies:** Redis (MULTI/EXEC), Lua scripts, sliding window counters

---

### Uber
**Patterns Used:**
- Geospatial: Redis GEORADIUS, S2 cells for driver-rider matching
- Dealing with Contention: Distributed locks for matching
- Real-time Updates: WebSocket for location updates, adaptive intervals
- Multi-step Processes: Temporal/Cadence for ride workflow

**Key Technologies:** Redis Geospatial, PostgreSQL/Cassandra, WebSocket, Temporal

---

### YouTube (Video Streaming)
**Patterns Used:**
- Handling Large Blobs: Adaptive bitrate streaming (HLS/DASH), CDN distribution
- Multi-step Processes: DAG pipeline (upload → transcode → thumbnail → index)
- Scaling Reads: Multi-quality transcoding, edge caching

**Key Technologies:** S3, CloudFront, FFmpeg (transcoding), async workers

---

### Tinder
**Patterns Used:**
- Geospatial: Geohashing for nearby profiles, consistent hashing for co-location
- Data Structures: Bloom filter to avoid showing same profile twice
- Pagination: Cursor-based feed with deduplication

**Key Technologies:** Redis Geospatial, Bloom filters, DynamoDB, cursor pagination

---

### LeetCode
**Patterns Used:**
- Multi-step Processes: Async code execution in sandboxed containers
- Scaling Writes: Queue-based execution (SQS), auto-scaling workers
- Security: Container isolation (Docker, seccomp, cgroups, resource limits)

**Key Technologies:** Docker, Kubernetes, SQS, serverless functions (Lambda)

---

### Web Crawler
**Patterns Used:**
- Multi-step Processes: DAG pipeline (fetch → parse → extract → store → index)
- Scaling Writes: Batching URL status updates
- Queue Management: SQS with DLQ, exponential backoff, robots.txt

**Key Technologies:** SQS, S3, Elasticsearch, distributed workers, politeness delays

---

### Ad Click Aggregator
**Patterns Used:**
- Stream Processing: Flink for real-time click counting
- Scaling Writes: Lambda architecture (batch + stream), reconciliation jobs
- Data Structures: Count-Min Sketch for heavy hitters
- Scaling Reads: OLAP pre-aggregation (ClickHouse)

**Key Technologies:** Flink, Kafka, ClickHouse, S3 (cold storage), reconciliation

---

### FB Post Search
**Patterns Used:**
- Search: Inverted index, Elasticsearch, bigrams for phrase search
- Scaling Writes: Batch index updates, cold storage for rare keywords
- Scaling Reads: Query result caching, index sharding

**Key Technologies:** Elasticsearch, Lucene, inverted indexes, write batching

---

## Quick Reference: Pattern Selection Guide

### When You Need...

**High Consistency (No Double-booking):**
→ ACID transactions, distributed locks, optimistic locking

**High Availability (Reads > Consistency):**
→ Multi-layer caching, read replicas, eventual consistency

**Real-time Updates:**
→ WebSocket (bi-directional), SSE (server→client), pub/sub

**Large File Storage:**
→ Presigned URLs, chunking/multipart, CDN, content-addressed storage

**Geospatial Queries:**
→ Redis Geospatial, geohashing, S2 geometry

**Text Search:**
→ Inverted index, Elasticsearch, autocomplete trie

**Unique IDs:**
→ Counter + Base62 (short URLs), UUID (distributed), Snowflake (sortable)

**Approximate Counting:**
→ Bloom filter (membership), Count-Min Sketch (frequency), HyperLogLog (cardinality)

**Multi-step Workflows:**
→ Temporal/Cadence (durable), Saga pattern (distributed), DAG pipelines

**High Write Throughput:**
→ Batching, stream processing (Flink/Kafka), sharding, async workers

**High Read Throughput:**
→ Caching (Redis/CDN), read replicas, pre-computation

---

## Conclusion

This pattern library consolidates battle-tested solutions from real-world systems at scale. When facing a system design problem:

1. **Identify the core challenges** (consistency vs availability, read vs write heavy, real-time needs)
2. **Match patterns to challenges** (use this guide as reference)
3. **Consider trade-offs** (every pattern has costs)
4. **Compose patterns** (real systems use multiple patterns together)
5. **Justify decisions** (explain why pattern X over pattern Y)

Remember: **There is no perfect solution**, only trade-offs appropriate for your specific requirements and constraints.

---

**Additional Resources:**
- HelloInterview Problem Breakdowns: https://www.hellointerview.com/learn/system-design/problem-breakdowns
- System Design Primer: https://github.com/donnemartin/system-design-primer
- Designing Data-Intensive Applications (Book): Martin Kleppmann

---

*This document synthesizes patterns from 16 comprehensive system design problems covering FAANG interview topics. Use it as a reference during interview preparation and system design discussions.*

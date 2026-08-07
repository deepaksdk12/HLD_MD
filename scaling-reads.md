# Scaling Reads — System Design Pattern

## Table of Contents

1. [The Problem](#the-problem)
2. [Why Reads Are Different From Writes](#why-reads-are-different-from-writes)
3. [Solution Progression](#solution-progression)
4. [Optimize Within Your Database](#optimize-within-your-database)
   - [Indexing](#indexing)
   - [Hardware Upgrades (Vertical Scaling)](#hardware-upgrades-vertical-scaling)
   - [Denormalization](#denormalization)
5. [Scale Your Database Horizontally](#scale-your-database-horizontally)
   - [Read Replicas](#read-replicas)
   - [Database Sharding](#database-sharding)
6. [Add External Caching Layers](#add-external-caching-layers)
   - [Application-Level Caching (Redis)](#application-level-caching-redis)
   - [CDN and Edge Caching](#cdn-and-edge-caching)
7. [Cache Patterns](#cache-patterns)
   - [Cache-Aside](#cache-aside)
   - [Write-Through](#write-through)
   - [Write-Behind](#write-behind)
   - [Read-Through](#read-through)
8. [Deep Dives](#deep-dives)
   - [Slow Queries as Dataset Grows](#slow-queries-as-dataset-grows)
   - [Hot Keys and Thundering Herd](#hot-keys-and-thundering-herd)
   - [Cache Stampede](#cache-stampede)
   - [Cache Invalidation](#cache-invalidation)
9. [Real-World Examples](#real-world-examples)
10. [Interview Cheat Sheet](#interview-cheat-sheet)

---

## The Problem

Read traffic grows far faster than write traffic in most applications.

```text
Instagram: post 1 photo (1 write) → 10,000 followers load feed (10,000 reads)
Twitter:   tweet once (1 write)   → 100M followers see it   (100M reads)
Amazon:    list 1 product (1 write) → 10,000 users browse  (10,000 reads)
YouTube:   upload 1 video (1 write) → 1B views served      (1B reads)
```

Typical read-to-write ratios:

| Application Type | Read:Write Ratio |
|-----------------|-----------------|
| Social feeds | 100:1 |
| E-commerce | 50:1 |
| News/content | 200:1 |
| General web apps | 10:1 |

When read volume spikes, your primary database becomes the bottleneck — and this is a **physics problem**, not a code problem. CPU, memory, and disk I/O are all finite resources.

---

## Why Reads Are Different From Writes

```text
Writes: must be consistent, atomic, durable — go to primary DB
Reads:  can be slightly stale, can be served from cache/replica

This asymmetry is the foundation of all read scaling strategies.
```

Key insight: **reads are idempotent** — reading the same data twice has no side effect. This makes them safe to cache, replicate, and serve from edge nodes.

---

## Solution Progression

Read scaling follows a natural progression:

```text
1. Optimize within your database (free, do this first)
        ↓ still slow?
2. Scale database horizontally (read replicas, sharding)
        ↓ still slow or too expensive?
3. Add external caching layers (Redis, CDN)
```

Each layer dramatically multiplies your read capacity.

---

## Optimize Within Your Database

### Indexing

The single biggest win before any architectural changes.

**Without index:** full table scan — O(n) per query

**With index:** B-tree lookup — O(log n) per query

```sql
-- Slow: full scan of 100M rows
SELECT * FROM posts WHERE user_id = 123 ORDER BY created_at DESC LIMIT 20;

-- Fast: index scan
CREATE INDEX idx_posts_user_created ON posts(user_id, created_at DESC);
```

**Spring Boot / JPA:**

```java
@Entity
@Table(name = "posts", indexes = {
    @Index(name = "idx_posts_user_created", columnList = "user_id, created_at DESC")
})
public class Post {
    @Id
    private Long id;
    private Long userId;
    private LocalDateTime createdAt;
}
```

**Rules for indexing:**

- Index columns used in `WHERE`, `ORDER BY`, `JOIN ON`
- Composite index column order matters — most selective column first
- Every index slows down writes (index must be updated) — don't over-index
- Covering index (includes all columns in SELECT) avoids table lookup entirely

```sql
-- Covering index — query satisfied entirely from index, no heap fetch
CREATE INDEX idx_cover ON posts(user_id, created_at DESC)
    INCLUDE (title, like_count); -- covers SELECT title, like_count
```

---

### Hardware Upgrades (Vertical Scaling)

Before adding complexity, throw better hardware at the problem.

| Resource | Upgrade | Effect |
|----------|---------|--------|
| RAM | More RAM | More data fits in buffer pool, fewer disk reads |
| CPU | More cores | More concurrent queries |
| Storage | SSD → NVMe | 10x faster random I/O |
| Network | Higher bandwidth | Faster data transfer |

**When to use:** Early stage, when architectural changes are premature.

**Limit:** Single machine has a ceiling. PostgreSQL on the largest AWS RDS instance tops out around 100K-500K simple reads/sec.

---

### Denormalization

Trade write complexity for read speed by storing precomputed or duplicated data.

**Normalized (slow reads):**

```sql
-- Need 3 joins to build a post card
SELECT p.title, u.name, u.avatar, COUNT(l.id) as like_count
FROM posts p
JOIN users u ON u.id = p.user_id
JOIN likes l ON l.post_id = p.id
WHERE p.id = 123
GROUP BY p.id, u.name, u.avatar;
```

**Denormalized (fast reads):**

```sql
-- Everything in one row, no joins
SELECT title, author_name, author_avatar, like_count
FROM post_feed_view
WHERE post_id = 123;
```

**Implementation with Spring Boot:**

```java
@Entity
@Table(name = "post_feed_view")
public class PostFeedView {
    @Id
    private Long postId;
    private String title;
    private String authorName;   // duplicated from users table
    private String authorAvatar; // duplicated from users table
    private int likeCount;       // precomputed aggregate
    private LocalDateTime createdAt;
}

// Update like_count on every like event (write cost)
@Transactional
public void addLike(Long postId, Long userId) {
    likeRepo.save(new Like(postId, userId));
    postFeedViewRepo.incrementLikeCount(postId); // maintain denormalized count
}
```

**Tradeoff:** Writes become more complex (must update denormalized fields), but reads are dramatically faster.

---

## Scale Your Database Horizontally

### Read Replicas

Add one or more **read-only copies** of the primary database. Writes go to primary, reads are distributed across replicas.

```text
               ┌─────────────────┐
   Writes ────▶│   Primary DB    │──replication──▶ Replica 1
               └─────────────────┘              └──▶ Replica 2
                                                └──▶ Replica 3

   Reads ────▶ Load Balancer ────▶ Replica 1 / 2 / 3
```

**Spring Boot with multiple DataSources:**

```java
@Configuration
public class DataSourceConfig {

    @Bean
    @Primary
    @ConfigurationProperties("spring.datasource.primary")
    public DataSource primaryDataSource() {
        return DataSourceBuilder.create().build(); // writes
    }

    @Bean
    @ConfigurationProperties("spring.datasource.replica")
    public DataSource replicaDataSource() {
        return DataSourceBuilder.create().build(); // reads
    }
}

// Route reads to replica using @Transactional(readOnly = true)
@Service
public class PostService {

    @Transactional(readOnly = true) // Spring routes to replica DataSource
    public List<Post> getFeed(Long userId) {
        return postRepo.findByUserId(userId);
    }

    @Transactional // goes to primary
    public Post createPost(PostRequest req) {
        return postRepo.save(new Post(req));
    }
}
```

**Replication lag:** Replicas are eventually consistent — typically 10ms-1s behind primary.

```text
User posts photo → primary updated
User immediately refreshes feed → reads from replica → might not see new photo yet
```

**Handling lag:**

- For critical reads (user's own data), read from primary
- For feed/discovery reads, stale data is acceptable
- Use `readYourWrites` routing: after a write, route subsequent reads to primary for N seconds

**When to use:** Read:write ratio > 5:1, writes are manageable on one primary

**Limit:** All replicas have the same full dataset — can't scale beyond one machine's storage

---

### Database Sharding

Split the dataset across multiple databases (shards), each owning a partition of the data.

```text
Shard 1: users 0–9M          Shard 2: users 9M–18M       Shard 3: users 18M–27M
┌─────────────────┐          ┌─────────────────┐          ┌─────────────────┐
│  Primary + 2    │          │  Primary + 2    │          │  Primary + 2    │
│  Replicas       │          │  Replicas       │          │  Replicas       │
└─────────────────┘          └─────────────────┘          └─────────────────┘
```

**Shard key selection (critical decision):**

```java
// Hash-based sharding — even distribution
int shardIndex = Math.abs(userId.hashCode()) % NUM_SHARDS;

// Range-based sharding — allows range queries on shard key
// userId 0-9M → Shard 1, 9M-18M → Shard 2...

// Directory-based sharding — flexible, uses lookup table
int shardIndex = shardLookupService.getShardForUser(userId);
```

**Spring Boot shard routing:**

```java
@Component
public class ShardRouter {

    private final Map<Integer, DataSource> shards;

    public DataSource getShardForUser(Long userId) {
        int shardIndex = (int)(userId % shards.size());
        return shards.get(shardIndex);
    }
}

@Service
public class PostService {

    public List<Post> getUserPosts(Long userId) {
        DataSource shard = shardRouter.getShardForUser(userId);
        return jdbcTemplate.query(shard, "SELECT * FROM posts WHERE user_id = ?", userId);
    }
}
```

**Hotspot problem:**

```text
hash(userId) % 3 shards
Celebrity user (100M followers) → all traffic to Shard 1 → Shard 1 is hot

Solutions:
  - Shard by content ID, not user ID
  - Over-shard (100 logical shards → 3 physical DBs, rebalance by moving shards)
  - Replicate hot shards more aggressively
```

**Cross-shard queries (expensive):**

```text
"Get trending posts across all users" → must query all shards, merge results
Solution: maintain a separate aggregation service / OLAP store for cross-shard queries
```

**When to use:** Dataset exceeds single machine capacity, writes also need to scale

---

## Add External Caching Layers

### Application-Level Caching (Redis)

Store frequently-read data in Redis (in-memory, microsecond reads) instead of hitting the DB.

```text
Request → Redis hit? → return cached data (microseconds)
        → Redis miss → DB query → store in Redis → return data
```

**Spring Boot with Spring Cache + Redis:**

```java
@Configuration
@EnableCaching
public class CacheConfig {

    @Bean
    public RedisCacheManager cacheManager(RedisConnectionFactory factory) {
        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
                .entryTtl(Duration.ofMinutes(10))
                .serializeValuesWith(
                    RedisSerializationContext.SerializationPair.fromSerializer(
                        new GenericJackson2JsonRedisSerializer()));
        return RedisCacheManager.builder(factory).cacheDefaults(config).build();
    }
}

@Service
public class PostService {

    @Cacheable(value = "posts", key = "#postId")
    public Post getPost(Long postId) {
        return postRepo.findById(postId) // only called on cache miss
                       .orElseThrow(() -> new NotFoundException(postId));
    }

    @CacheEvict(value = "posts", key = "#post.id")
    public Post updatePost(Post post) {
        return postRepo.save(post); // evicts stale cache entry
    }

    @Cacheable(value = "user-feed", key = "#userId", unless = "#result.isEmpty()")
    public List<PostSummary> getUserFeed(Long userId) {
        return postRepo.findFeedForUser(userId);
    }
}
```

**Manual Redis operations (more control):**

```java
@Service
public class ScorecardService {

    private final StringRedisTemplate redisTemplate;
    private final ObjectMapper objectMapper;

    public Scorecard getScorecard(String matchId) {
        String key = "scorecard:" + matchId;
        String cached = redisTemplate.opsForValue().get(key);

        if (cached != null) {
            return objectMapper.readValue(cached, Scorecard.class);
        }

        Scorecard data = db.fetchScorecard(matchId);
        long ttl = 60 + ThreadLocalRandom.current().nextInt(10); // jittered TTL
        redisTemplate.opsForValue().set(key, objectMapper.writeValueAsString(data),
                Duration.ofSeconds(ttl));
        return data;
    }
}
```

**What to cache:**

| Good Candidates | Bad Candidates |
|----------------|----------------|
| User profiles | Financial balances |
| Product listings | Inventory counts |
| Feed/timeline | Authentication tokens |
| Config/feature flags | Real-time prices |
| Aggregated counts (likes) | Write-heavy data |

---

### CDN and Edge Caching

Push static and semi-static content to **edge nodes** geographically close to users.

```text
User in Mumbai ──▶ CDN Edge (Mumbai) ──cache hit──▶ serve from edge (5ms)
                                      ──cache miss──▶ Origin Server (200ms)
                                                   ──▶ cache in edge for next user
```

**What CDNs cache:**

- Static assets (JS, CSS, images, fonts)
- API responses with long TTL (product catalog, public profiles)
- Video/audio streams
- HTML pages (for static sites)

**Cache-Control headers (Spring Boot):**

```java
@RestController
public class ProductController {

    @GetMapping("/products/{id}")
    public ResponseEntity<Product> getProduct(@PathVariable Long id) {
        Product product = productService.getProduct(id);
        return ResponseEntity.ok()
                .cacheControl(CacheControl.maxAge(300, TimeUnit.SECONDS)  // CDN caches for 5 min
                                         .staleWhileRevalidate(60, TimeUnit.SECONDS))
                .body(product);
    }

    @GetMapping("/users/{id}/avatar")
    public ResponseEntity<byte[]> getAvatar(@PathVariable Long id) {
        byte[] image = userService.getAvatar(id);
        return ResponseEntity.ok()
                .cacheControl(CacheControl.maxAge(7, TimeUnit.DAYS).cachePublic()) // CDN caches for 7 days
                .body(image);
    }
}
```

**CDN invalidation on update:**

```java
@Service
public class ProductService {

    private final CdnClient cdnClient;

    @Transactional
    public Product updateProduct(Long id, ProductRequest req) {
        Product product = productRepo.save(new Product(id, req));
        cdnClient.invalidate("/products/" + id); // purge stale CDN cache
        return product;
    }
}
```

---

## Cache Patterns

### Cache-Aside

Application manages cache explicitly. Most flexible, most common.

```text
Read:  check cache → miss → read DB → write to cache → return
Write: write DB → invalidate/update cache
```

```java
public Post getPost(Long postId) {
    Post cached = redis.get("post:" + postId);
    if (cached != null) return cached;

    Post post = db.findPost(postId);
    redis.set("post:" + postId, post, Duration.ofMinutes(10));
    return post;
}
```

**Pros:** Cache only what's actually needed, resilient (app works if cache is down)
**Cons:** Cache miss adds latency, potential for stale data

---

### Write-Through

Every write goes to both cache and DB simultaneously.

```text
Write: write cache → write DB → return
Read:  check cache → always fresh (if write-through is consistent)
```

```java
@Transactional
public Post createPost(PostRequest req) {
    Post post = db.save(new Post(req));
    redis.set("post:" + post.getId(), post, Duration.ofMinutes(10)); // write to cache too
    return post;
}
```

**Pros:** Cache always fresh, no cold start problem
**Cons:** Every write hits both cache and DB — higher write latency, cache fills with rarely-read data

---

### Write-Behind (Write-Back)

Write to cache immediately, DB write happens asynchronously.

```text
Write: write cache → return (fast) → background thread writes to DB
Read:  always served from cache
```

```java
public void updateLikeCount(Long postId, int delta) {
    redis.incrBy("likes:" + postId, delta); // instant
    asyncQueue.enqueue(new LikeCountUpdate(postId, delta)); // DB write later
}
```

**Pros:** Lowest write latency
**Cons:** Risk of data loss if cache fails before DB write, complex consistency

**When to use:** High-write counters (like counts, view counts, inventory) where losing a few updates is acceptable

---

### Read-Through

Cache sits in front of DB; cache populates itself on miss.

```text
Read: check cache → miss → cache fetches from DB → cache returns to app
```

Typically implemented by the cache library (e.g., Ehcache, Caffeine with loader):

```java
LoadingCache<Long, Post> cache = Caffeine.newBuilder()
        .maximumSize(100_000)
        .expireAfterWrite(10, TimeUnit.MINUTES)
        .build(postId -> db.findPost(postId)); // auto-loads on miss

public Post getPost(Long postId) {
    return cache.get(postId); // automatically loads from DB if not cached
}
```

**Pros:** Simple application code, consistent loading logic
**Cons:** First request always slow (cache miss), less control than cache-aside

---

## Deep Dives

### Slow Queries as Dataset Grows

**Problem:** Query was fast with 1M rows, now slow with 100M rows.

**Diagnosis:**

```sql
EXPLAIN ANALYZE SELECT * FROM posts WHERE user_id = 123 ORDER BY created_at DESC LIMIT 20;
-- Look for: Seq Scan (bad) vs Index Scan (good)
-- Look for: actual rows >> estimated rows (stale statistics)
```

**Solutions in order of effort:**

1. Add missing index (fastest fix)
2. Update statistics: `ANALYZE posts;`
3. Rewrite query to use index (avoid functions on indexed columns)
4. Pagination with keyset instead of OFFSET

```java
// BAD — OFFSET 10000 still scans 10000 rows
Page<Post> page = postRepo.findByUserId(userId, PageRequest.of(500, 20)); // OFFSET 10000

// GOOD — keyset pagination, always O(log n) regardless of page number
List<Post> posts = postRepo.findByUserIdAndCreatedAtBefore(
        userId, lastSeenCreatedAt, PageRequest.of(0, 20));
```

---

### Hot Keys and Thundering Herd

**Problem:** One key (e.g., a celebrity's profile) gets 1M reads/second — overwhelms even Redis.

**Solutions:**

1. **Local in-process cache** — JVM-level cache before Redis

```java
// Caffeine local cache in front of Redis
private final Cache<Long, UserProfile> localCache = Caffeine.newBuilder()
        .maximumSize(1000)
        .expireAfterWrite(5, TimeUnit.SECONDS) // very short TTL for hot data
        .build();

public UserProfile getProfile(Long userId) {
    return localCache.get(userId, id -> {
        UserProfile cached = redis.get("profile:" + id);
        if (cached != null) return cached;
        return db.findUser(id);
    });
}
```

2. **Key replication** — store same value under multiple keys, read from random one

```java
// Write to N replicated keys
IntStream.range(0, REPLICA_COUNT).forEach(i ->
        redis.set("profile:" + userId + ":" + i, profile, Duration.ofMinutes(10)));

// Read from random replica
int replica = ThreadLocalRandom.current().nextInt(REPLICA_COUNT);
return redis.get("profile:" + userId + ":" + replica);
```

3. **Read-through with jittered TTL** — prevent all replicas expiring at once

```java
long ttl = 600 + ThreadLocalRandom.current().nextInt(60); // 600-660s
redis.set(key, value, Duration.ofSeconds(ttl));
```

---

### Cache Stampede

**Problem:** A popular cached item expires. Thousands of requests simultaneously miss the cache, all hit the DB at once → DB overwhelmed.

```text
Cache expires at T=0
T=0: 10,000 requests → all miss cache → all query DB → DB dies
```

**Solution 1: Probabilistic early expiration**

Refresh the cache slightly before it actually expires, based on probability:

```java
public Post getPost(Long postId) {
    CachedItem item = redis.getWithTtl("post:" + postId);

    if (item != null) {
        // probabilistically refresh early if TTL is getting low
        double remainingFraction = (double) item.getTtlSeconds() / ORIGINAL_TTL;
        if (remainingFraction < 0.1 && Math.random() < 0.1) {
            asyncRefresh(postId); // background refresh, return stale value now
        }
        return item.getValue();
    }

    return loadAndCache(postId);
}
```

**Solution 2: Mutex lock (cache stampede prevention)**

```java
public Post getPost(Long postId) {
    Post cached = redis.get("post:" + postId);
    if (cached != null) return cached;

    String lockKey = "lock:post:" + postId;
    Boolean locked = redis.setIfAbsent(lockKey, "1", Duration.ofSeconds(5));

    if (Boolean.TRUE.equals(locked)) {
        // this thread rebuilds the cache
        try {
            Post post = db.findPost(postId);
            redis.set("post:" + postId, post, Duration.ofMinutes(10));
            return post;
        } finally {
            redis.delete(lockKey);
        }
    } else {
        // another thread is rebuilding — wait briefly and retry
        Thread.sleep(50);
        return getPost(postId); // recursive retry
    }
}
```

**Solution 3: Stale-while-revalidate**

Serve stale data while refreshing in background — never have a cache miss delay:

```java
// Cache stores value + expiry metadata
public Post getPost(Long postId) {
    CachedPost cached = redis.get("post:" + postId);

    if (cached != null && !cached.isExpired()) {
        return cached.getPost();
    }

    if (cached != null && cached.isSoftExpired()) {
        // serve stale, refresh in background
        CompletableFuture.runAsync(() -> refreshCache(postId));
        return cached.getPost();
    }

    return loadAndCache(postId);
}
```

---

### Cache Invalidation

**"There are only two hard things in Computer Science: cache invalidation and naming things."**

**Problem:** Data changes in DB, but cache still serves stale version.

**Strategies:**

| Strategy | How | Consistency | Complexity |
|----------|-----|-------------|------------|
| TTL expiry | Cache expires after N seconds | Eventual (stale for TTL duration) | Lowest |
| Write invalidation | On write, delete cache key | Strong | Low |
| Write update | On write, update cache value | Strong | Medium |
| Event-driven | Kafka event triggers invalidation | Near-strong | High |

**Write invalidation (most common):**

```java
@Service
public class PostService {

    @Transactional
    public Post updatePost(Long postId, PostRequest req) {
        Post post = postRepo.save(new Post(postId, req));
        redis.delete("post:" + postId);          // invalidate single post
        redis.delete("user-feed:" + post.getUserId()); // invalidate feed containing this post
        return post;
    }
}
```

**Event-driven invalidation (for distributed systems):**

```java
// Post service publishes update event
@Transactional
public Post updatePost(Long postId, PostRequest req) {
    Post post = postRepo.save(new Post(postId, req));
    kafkaTemplate.send("post-updates", new PostUpdatedEvent(postId));
    return post;
}

// Cache service listens and invalidates
@KafkaListener(topics = "post-updates")
public void onPostUpdated(PostUpdatedEvent event) {
    redis.delete("post:" + event.getPostId());
    // invalidate all feeds that contained this post — expensive!
}
```

**Cache invalidation for fan-out reads (hardest case):**

```text
User updates profile photo
→ must invalidate: their profile cache, all feed entries showing them, all comment sections
→ potentially millions of cache keys to invalidate
```

**Solution: version tags**

```java
// Store a version number, include it in cache keys
public UserProfile getProfile(Long userId) {
    long version = redis.get("user-version:" + userId); // cheap lookup
    return redis.get("profile:" + userId + ":v" + version);
}

// On update: bump version (all old cache keys become unreferenced and expire naturally)
public void updateProfile(Long userId, ProfileRequest req) {
    db.updateProfile(userId, req);
    redis.incr("user-version:" + userId); // atomically bump version
    // old "profile:123:v5" will expire via TTL — no need to find and delete
}
```

---

## Real-World Examples

### Instagram Feed

```text
Read path (100M reads/day):

User opens app
    │
    ▼
CDN edge → serve static assets (JS, CSS, images) from edge
    │
    ▼
API Gateway → Feed Service
    │
    ▼
Redis (L1): "feed:userId:123" → pre-computed feed (list of post IDs)
    │ miss
    ▼
Feed DB Replica → fetch post IDs for user's feed
    │
    ▼
Redis pipeline: fetch post details for each post ID (batch get)
    │
    ▼
Return to client

Write path (1M posts/day):
  New post → Fan-out service → push to followers' feed caches
           → invalidate stale feed caches for active users
```

- **CDN:** Images/videos cached at edge (99% of bandwidth)
- **Redis L1:** Pre-computed feed lists (TTL 1h)
- **Read replicas:** 10 replicas per primary shard
- **Denormalization:** `post_feed_view` table — no joins at read time

---

### YouTube Top Videos

```text
Read: billions of views/day

Video metadata → cached in Redis (title, description, like count)
Video stream   → served entirely from CDN (no origin hit for cached videos)
View count     → write-behind: Redis counter (instant) → batch DB sync (every minute)
Recommendations → pre-computed offline, stored in Redis sorted sets

Key insight: 80% of views are on 20% of videos
  → cache those 20% aggressively
  → CDN pre-warms popular videos to all edge nodes
```

---

### Ticketmaster Seat Availability

```text
Read: millions of users checking seat availability simultaneously
Write: seats change as tickets are purchased (high contention)

Strategy:
  - Redis stores seat availability map (bitset per event)
  - Reads served entirely from Redis (microseconds)
  - Writes go to primary DB + publish to Redis via pub/sub
  - Short TTL (5s) — stale data acceptable for browsing, not for checkout

On high-demand sale:
  - Read replicas absorb browsing traffic
  - Primary DB only handles checkout (write path)
  - Redis cluster sharded by eventId
```

---

## Interview Cheat Sheet

### When to Use Which Strategy

| Problem | Solution |
|---------|---------|
| Queries slow with growing data | Add indexes, keyset pagination |
| Read:write ratio 5:1 | Read replicas |
| Dataset exceeds one machine | Database sharding |
| Same data read millions of times | Redis cache (cache-aside) |
| Static/semi-static content | CDN with long TTL |
| High-write counters (likes, views) | Write-behind (Redis counter + async DB) |
| Celebrity/hot key problem | Local in-process cache + key replication |
| Cache stampede | Mutex lock or stale-while-revalidate |
| Cache invalidation at scale | Version tags or event-driven invalidation |

---

### Cache Pattern Comparison

| Pattern | Read Latency | Write Latency | Consistency | Complexity |
|---------|-------------|---------------|-------------|------------|
| Cache-Aside | Miss: DB latency | DB only | Eventual | Low |
| Write-Through | Always fast | Cache + DB | Strong | Medium |
| Write-Behind | Always fast | Cache only (fast) | Eventual | High |
| Read-Through | Miss: DB latency | DB only | Eventual | Low |

---

### Common Deep Dive Answers

**"What happens when queries start taking longer as your dataset grows?"**

- Run `EXPLAIN ANALYZE` to find sequential scans
- Add composite index matching query pattern
- Switch from OFFSET pagination to keyset pagination
- Denormalize if joins are the bottleneck
- If dataset > single machine: shard

**"How do you handle millions of concurrent reads for the same cached data?"**

- L1 local cache (Caffeine) in each JVM instance — absorbs most traffic
- Redis replica reads — distribute across Redis cluster
- Hot key replication — store same value under multiple Redis keys

**"What happens when multiple requests try to rebuild an expired cache entry simultaneously?"**

- Mutex lock in Redis (SETNX) — only one rebuilds, others wait
- Stale-while-revalidate — serve stale data, refresh in background
- Probabilistic early expiration — refresh before expiry, avoid cliff

**"How do you handle cache invalidation when data updates must be immediately visible?"**

- Write invalidation: delete cache key on every write
- Event-driven: Kafka event triggers invalidation across all nodes
- Version tags: bump version on write, old cache keys expire naturally
- If truly must be real-time: skip cache, read from primary DB replica

---

### Architecture Decision Flowchart

```text
Reads are slow or DB is overloaded?
          │
          ▼
Have you added indexes?
   NO → Add indexes first (free, biggest win)
   YES → continue
          │
          ▼
Read:write ratio > 5:1?
   YES → Add read replicas
   NO  → continue
          │
          ▼
Dataset exceeds single machine capacity?
   YES → Shard the database
   NO  → continue
          │
          ▼
Same data read repeatedly across users?
   YES → Redis cache (cache-aside with TTL)
   NO  → continue
          │
          ▼
Content is static or rarely changes?
   YES → CDN caching with long TTL
   NO  → Short TTL + write invalidation
          │
          ▼
High-write counters (likes, views)?
   YES → Write-behind (Redis counter, async DB sync)
```

---

### Key Terms Quick Reference

| Term | Meaning |
|------|---------|
| **Cache-aside** | App checks cache, falls back to DB on miss |
| **Write-through** | Write to cache and DB simultaneously |
| **Write-behind** | Write to cache instantly, DB asynchronously |
| **TTL** | Time-to-live — how long a cache entry is valid |
| **Cache stampede** | Many simultaneous misses on same expired key |
| **Hot key** | Single cache key receiving extreme traffic |
| **Read replica** | Read-only copy of primary DB |
| **Shard** | Horizontal partition of database by key range or hash |
| **Denormalization** | Storing redundant data to avoid joins |
| **Covering index** | Index that includes all columns needed by a query |
| **Keyset pagination** | Paginate by last-seen value instead of OFFSET |
| **Stale-while-revalidate** | Serve stale data while refreshing in background |

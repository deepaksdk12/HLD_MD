# Scaling Writes — System Design Pattern

## Table of Contents

1. [The Challenge](#the-challenge)
2. [Why Writes Are Harder Than Reads](#why-writes-are-harder-than-reads)
3. [Solution Overview](#solution-overview)
4. [Vertical Scaling and Write Optimization](#vertical-scaling-and-write-optimization)
   - [Vertical Scaling](#vertical-scaling)
   - [Database Choices](#database-choices)
   - [Write Optimization Within the DB](#write-optimization-within-the-db)
5. [Sharding and Partitioning](#sharding-and-partitioning)
   - [Horizontal Sharding](#horizontal-sharding)
   - [Vertical Partitioning](#vertical-partitioning)
6. [Handling Bursts with Queues and Load Shedding](#handling-bursts-with-queues-and-load-shedding)
   - [Write Queues for Burst Handling](#write-queues-for-burst-handling)
   - [Load Shedding Strategies](#load-shedding-strategies)
7. [Batching and Hierarchical Aggregation](#batching-and-hierarchical-aggregation)
   - [Batching](#batching)
   - [Hierarchical Aggregation](#hierarchical-aggregation)
8. [Deep Dives](#deep-dives)
   - [Resharding Without Downtime](#resharding-without-downtime)
   - [Hot Key Problem](#hot-key-problem)
9. [Real-World Examples](#real-world-examples)
10. [Interview Cheat Sheet](#interview-cheat-sheet)

---

## The Challenge

Write scaling is harder than read scaling because writes have fundamental constraints reads don't:

```text
Reads:  idempotent, cacheable, distributable to replicas
Writes: must be durable, consistent, often ordered — can't simply cache or replicate away
```

Real scenarios that break naive write designs:

```text
Ad click tracker:    10M clicks/second during Super Bowl ad
Metrics ingestor:    100K time-series data points/second from IoT devices
Strava segment:      50K users finish the same cycling segment simultaneously → leaderboard contention
YouTube view count:  1M concurrent viewers hitting the same counter
Rate limiter:        100K requests/second checking and updating counters
```

In all these cases, the bottleneck is the same: a single write target (DB, row, counter) that can only handle so many writes per second.

---

## Why Writes Are Harder Than Reads

| Constraint | Reads | Writes |
|------------|-------|--------|
| Cacheable | Yes | No (must persist) |
| Distributable | Yes (replicas) | Limited (primary only for strong consistency) |
| Idempotent | Yes | Often no (double-write = double charge) |
| Ordered | Rarely matters | Often critical |
| Contention | Rarely | High (hot rows, locks) |
| Disk I/O | Sequential OK | Random I/O is bottleneck |

The core physics constraint:

```text
SSD random write: ~100K IOPS
PostgreSQL on large RDS: ~50K–100K simple writes/sec
Beyond this → must shard or use alternative strategies
```

---

## Solution Overview

Four strategies, applied in order of complexity:

```text
1. Vertical Scaling + DB Choice    (stay on one machine, optimize it)
        ↓ still bottlenecked?
2. Sharding and Partitioning       (split data across machines)
        ↓ write bursts overwhelm even shards?
3. Queues + Load Shedding          (absorb bursts, protect the DB)
        ↓ aggregating many writes to same value?
4. Batching + Hierarchical Aggregation  (collapse many writes into fewer)
```

---

## Vertical Scaling and Write Optimization

### Vertical Scaling

Before any architectural change, squeeze more out of a single machine.

| Upgrade | Effect on Writes |
|---------|-----------------|
| More RAM | Larger write buffer — more writes absorbed before flush |
| NVMe SSD | 5–10x faster random I/O vs SATA SSD |
| More CPU cores | More concurrent write transactions |
| Faster network | Reduces replication lag to replicas |

**Limit:** A single machine tops out. The largest AWS RDS instance (`db.r8g.48xlarge`) is still one machine with one disk controller.

**When to use:** Early stage. Vertical scaling is cheap and fast — exhaust it before adding complexity.

---

### Database Choices

Different databases handle writes differently. Choosing the right DB is the highest-leverage early decision.

#### Write-Optimized Databases

**LSM-Tree databases (Cassandra, ScyllaDB, RocksDB):**

```text
Write path:
  1. Write to in-memory MemTable (fast, ~microseconds)
  2. Append to Write-Ahead Log (WAL) on disk for durability
  3. Background: flush MemTable to SSTable on disk

No random I/O on writes — everything is sequential append
→ 10–100x write throughput vs B-tree databases at scale
```

```text
B-Tree databases (PostgreSQL, MySQL):
  Write → update B-tree in place → random disk I/O → slow at scale

LSM databases (Cassandra, RocksDB):
  Write → append to log → sequential I/O → fast at scale
  Read → slower (must merge multiple SSTables)
```

**Spring Boot with Cassandra:**

```java
@Table("ad_clicks")
public class AdClick {
    @PrimaryKeyColumn(type = PrimaryKeyType.PARTITIONED)
    private String adId;

    @PrimaryKeyColumn(type = PrimaryKeyType.CLUSTERED)
    private UUID clickId;

    private String userId;
    private Instant clickedAt;
}

@Repository
public interface AdClickRepository extends CassandraRepository<AdClick, UUID> {}

@Service
public class AdClickService {

    private final AdClickRepository clickRepo;

    public void recordClick(String adId, String userId) {
        AdClick click = new AdClick();
        click.setAdId(adId);
        click.setClickId(UUIDs.timeBased());
        click.setUserId(userId);
        click.setClickedAt(Instant.now());
        clickRepo.save(click); // sequential write, very fast
    }
}
```

#### Choosing the Right DB for Write Patterns

| Write Pattern | Best DB Choice | Why |
|--------------|----------------|-----|
| High-throughput append | Cassandra, ScyllaDB | LSM-tree, sequential writes |
| Time-series metrics | InfluxDB, TimescaleDB | Optimized for timestamped inserts |
| Event streaming | Kafka | Append-only log, massive throughput |
| Counters / aggregates | Redis | In-memory, atomic increments |
| Transactional writes | PostgreSQL, MySQL | ACID, strong consistency |
| Document writes | MongoDB | Flexible schema, write concern tuning |

---

### Write Optimization Within the DB

Before sharding, tune the DB for writes.

**PostgreSQL tuning for writes:**

```sql
-- Disable synchronous commit for non-critical writes (10x throughput gain, small durability risk)
SET synchronous_commit = off;

-- Bulk insert instead of row-by-row
INSERT INTO events (user_id, event_type, created_at)
VALUES (1, 'click', NOW()), (2, 'view', NOW()), (3, 'click', NOW()); -- batch insert

-- Use COPY for very large bulk loads (fastest)
COPY events FROM '/tmp/events.csv' CSV;

-- Partial indexes — smaller, faster to update
CREATE INDEX idx_active_users ON users(id) WHERE status = 'active';
```

**Spring Boot bulk insert (JPA batch):**

```java
# application.properties
spring.jpa.properties.hibernate.jdbc.batch_size=100
spring.jpa.properties.hibernate.order_inserts=true
spring.jpa.properties.hibernate.order_updates=true

@Service
public class EventService {

    @Transactional
    public void bulkInsert(List<Event> events) {
        // Hibernate sends as one batch of 100 INSERTs, not 100 individual statements
        eventRepo.saveAll(events);
    }
}
```

**Avoid write contention:**

```java
// BAD — SELECT then UPDATE causes race condition under concurrency
long count = counterRepo.getCount(adId);
counterRepo.setCount(adId, count + 1); // lost update if two threads do this simultaneously

// GOOD — atomic increment in DB
@Modifying
@Query("UPDATE AdCounter c SET c.count = c.count + 1 WHERE c.adId = :adId")
void incrementCount(@Param("adId") String adId);

// BETTER for extreme write rates — delegate to Redis
redisTemplate.opsForValue().increment("ad:count:" + adId);
```

---

## Sharding and Partitioning

### Horizontal Sharding

Split rows across multiple database instances based on a shard key.

```text
Without sharding:          With sharding (3 shards):
┌─────────────┐           ┌──────────┐ ┌──────────┐ ┌──────────┐
│  All writes │           │ Shard 1  │ │ Shard 2  │ │ Shard 3  │
│  → 1 DB     │           │ adId A-H │ │ adId I-P │ │ adId Q-Z │
│  50K wps    │           │ 50K wps  │ │ 50K wps  │ │ 50K wps  │
└─────────────┘           └──────────┘ └──────────┘ └──────────┘
                          Total: 150K writes/sec
```

**Shard key selection — the most critical decision:**

```text
Good shard key:   high cardinality, evenly distributed, aligns with access pattern
Bad shard key:    low cardinality, skewed distribution, causes cross-shard queries

Examples:
  Ad clicks:     shard by adId      → clicks for one ad go to one shard
  User events:   shard by userId    → all user's data co-located
  Time-series:   shard by timerange → potential hot shard on current time window
  Global counter: no good shard key → use different strategy (Redis, aggregation)
```

**Spring Boot shard routing:**

```java
@Configuration
public class ShardDataSourceConfig {

    private final Map<Integer, DataSource> shards = new HashMap<>();

    @PostConstruct
    public void init() {
        shards.put(0, buildDataSource("jdbc:postgresql://shard0:5432/db"));
        shards.put(1, buildDataSource("jdbc:postgresql://shard1:5432/db"));
        shards.put(2, buildDataSource("jdbc:postgresql://shard2:5432/db"));
    }

    public DataSource getShardForKey(String shardKey) {
        int shardIndex = Math.abs(shardKey.hashCode()) % shards.size();
        return shards.get(shardIndex);
    }
}

@Service
public class AdClickService {

    private final ShardDataSourceConfig shardConfig;
    private final JdbcTemplate jdbcTemplate;

    public void recordClick(String adId, String userId) {
        DataSource shard = shardConfig.getShardForKey(adId);
        jdbcTemplate.update(shard,
            "INSERT INTO ad_clicks (ad_id, user_id, clicked_at) VALUES (?, ?, ?)",
            adId, userId, Instant.now());
    }
}
```

**Consistent hashing (for dynamic shard counts):**

```java
@Component
public class ConsistentHashRouter {

    private final TreeMap<Integer, String> ring = new TreeMap<>();
    private static final int VIRTUAL_NODES = 150; // more = better distribution

    public void addShard(String shardId) {
        for (int i = 0; i < VIRTUAL_NODES; i++) {
            int hash = hash(shardId + ":" + i);
            ring.put(hash, shardId);
        }
    }

    public String getShardForKey(String key) {
        int hash = hash(key);
        Map.Entry<Integer, String> entry = ring.ceilingEntry(hash);
        return entry != null ? entry.getValue() : ring.firstEntry().getValue();
    }

    private int hash(String key) {
        return key.hashCode() & 0x7FFFFFFF;
    }
}
```

---

### Vertical Partitioning

Split columns across tables or databases — separate hot columns from cold columns.

```text
Before (one wide table — all columns written on every update):
  users: id | name | email | bio | avatar_url | last_login | settings_json | ...

After (separate hot from cold):
  users_core:     id | name | email | last_login     (written frequently)
  users_profile:  id | bio | avatar_url              (written rarely)
  users_settings: id | settings_json                 (written rarely)
```

```java
@Entity
@Table(name = "users_core")
public class UserCore {
    @Id
    private Long id;
    private String name;
    private String email;
    private Instant lastLogin; // updated on every login — hot column
}

@Entity
@Table(name = "users_profile")
public class UserProfile {
    @Id
    private Long userId;
    private String bio;
    private String avatarUrl; // updated rarely — cold column
}
```

**Effect:** Writes to the hot table are smaller, fit more rows per page, less lock contention.

---

## Handling Bursts with Queues and Load Shedding

### Write Queues for Burst Handling

Instead of writing directly to the DB, writes go to a queue (Kafka). DB consumers process at a sustainable rate.

```text
Without queue:                    With queue:
                                  
Burst: 1M writes/sec             Burst: 1M writes/sec
         │                                │
         ▼                                ▼
    Primary DB              Kafka Topic (absorbs burst)
    (max 50K wps)                         │
    → DB overloaded                (processes at 50K wps)
    → writes fail                         ▼
                                    Consumer → DB
                                    (steady 50K wps)
                                    → DB healthy
```

**Spring Boot with Kafka producer (write side):**

```java
@Service
public class AdClickService {

    private final KafkaTemplate<String, AdClickEvent> kafkaTemplate;

    public CompletableFuture<Void> recordClick(String adId, String userId) {
        AdClickEvent event = new AdClickEvent(adId, userId, Instant.now());

        return kafkaTemplate.send("ad-clicks", adId, event) // adId as key → same partition
                .thenApply(result -> null)
                .exceptionally(ex -> {
                    log.error("Failed to enqueue click for adId={}", adId, ex);
                    return null;
                });
    }
}
```

**Spring Boot with Kafka consumer (DB write side):**

```java
@Component
public class AdClickConsumer {

    private final AdClickRepository clickRepo;

    @KafkaListener(
        topics = "ad-clicks",
        groupId = "ad-click-writers",
        containerFactory = "batchKafkaListenerContainerFactory" // batch mode
    )
    public void consumeBatch(List<ConsumerRecord<String, AdClickEvent>> records) {
        List<AdClick> clicks = records.stream()
                .map(r -> new AdClick(r.value()))
                .collect(Collectors.toList());

        clickRepo.saveAll(clicks); // one batch INSERT for entire batch
    }
}

// Batch listener factory
@Bean
public KafkaListenerContainerFactory<?> batchKafkaListenerContainerFactory(
        ConsumerFactory<String, AdClickEvent> consumerFactory) {

    ConcurrentKafkaListenerContainerFactory<String, AdClickEvent> factory =
            new ConcurrentKafkaListenerContainerFactory<>();
    factory.setConsumerFactory(consumerFactory);
    factory.setBatchListener(true);
    factory.getContainerProperties().setPollTimeout(3000);
    return factory;
}
```

**Benefits of Kafka as write buffer:**

```text
Durability:    Kafka persists messages — no data loss during DB maintenance
Backpressure:  DB can process at its own pace, no bursts passed through
Replay:        Reprocess events if consumer has a bug
Decoupling:    Write path and DB write are independently scalable
Partitioning:  adId as key ensures same ad's clicks go to same partition (ordered)
```

---

### Load Shedding Strategies

When the queue itself is overwhelmed, or for use cases where dropping some writes is acceptable.

#### Rate Limiting

```java
@Component
public class RateLimiter {

    private final StringRedisTemplate redisTemplate;

    public boolean allowWrite(String key, int maxPerSecond) {
        String counterKey = "rate:" + key + ":" + Instant.now().getEpochSecond();

        Long count = redisTemplate.opsForValue().increment(counterKey);
        redisTemplate.expire(counterKey, Duration.ofSeconds(2));

        return count <= maxPerSecond;
    }
}

@Service
public class AdClickService {

    public void recordClick(String adId, String userId) {
        if (!rateLimiter.allowWrite("ad:" + adId, 10_000)) {
            // shed this write — ad is being clicked too fast (likely bot)
            metrics.increment("clicks.rate_limited");
            return;
        }
        // proceed with write
    }
}
```

#### Circuit Breaker

```java
// Using Resilience4j
@Service
public class AdClickService {

    @CircuitBreaker(name = "db-write", fallbackMethod = "queueForLater")
    public void recordClick(String adId, String userId) {
        clickRepo.save(new AdClick(adId, userId));
    }

    // Fallback: enqueue when DB is struggling
    public void queueForLater(String adId, String userId, Exception ex) {
        kafkaTemplate.send("ad-clicks-overflow", adId, new AdClickEvent(adId, userId));
    }
}
```

#### Token Bucket (smooth burst handling)

```java
@Component
public class TokenBucketRateLimiter {

    private final StringRedisTemplate redisTemplate;

    private static final String LUA_SCRIPT =
        "local tokens = tonumber(redis.call('get', KEYS[1]) or ARGV[1]) " +
        "local now = tonumber(ARGV[2]) " +
        "local last = tonumber(redis.call('get', KEYS[2]) or now) " +
        "local delta = math.max(0, now - last) " +
        "local refill = delta * tonumber(ARGV[3]) " +
        "tokens = math.min(tonumber(ARGV[1]), tokens + refill) " +
        "if tokens >= 1 then tokens = tokens - 1 " +
        "  redis.call('set', KEYS[1], tokens) " +
        "  redis.call('set', KEYS[2], now) " +
        "  return 1 " +
        "else return 0 end";

    public boolean tryConsume(String key) {
        List<String> keys = List.of("bucket:" + key, "bucket:ts:" + key);
        Long result = redisTemplate.execute(
            RedisScript.of(LUA_SCRIPT, Long.class),
            keys,
            "100",                          // max tokens
            String.valueOf(System.currentTimeMillis() / 1000),
            "10"                            // refill rate: 10 tokens/sec
        );
        return Long.valueOf(1L).equals(result);
    }
}
```

---

## Batching and Hierarchical Aggregation

### Batching

Accumulate writes in memory or a buffer, then flush to the DB as one batch operation.

**In-memory batch with scheduled flush:**

```java
@Component
public class AdClickBatcher {

    private final ConcurrentLinkedQueue<AdClick> buffer = new ConcurrentLinkedQueue<>();
    private final AdClickRepository clickRepo;

    public void add(AdClick click) {
        buffer.offer(click); // non-blocking, microseconds
    }

    @Scheduled(fixedDelay = 1000) // flush every second
    @Transactional
    public void flush() {
        if (buffer.isEmpty()) return;

        List<AdClick> batch = new ArrayList<>();
        AdClick item;
        while ((item = buffer.poll()) != null && batch.size() < 10_000) {
            batch.add(item);
        }

        if (!batch.isEmpty()) {
            clickRepo.saveAll(batch); // one batch INSERT for up to 10K rows
            metrics.record("batch.size", batch.size());
        }
    }
}
```

**Tradeoff:** Writes are durable only after flush. Crash before flush = data in buffer is lost.

**Mitigation:** Write to Kafka first (durable), batch consumer flushes to DB:

```text
API → Kafka (durable, instant) → Consumer → batched DB INSERT (every second)
                                                     ↑
                              no data loss: Kafka holds messages until DB ack
```

---

### Hierarchical Aggregation

For high-write counters (view counts, like counts, click totals), aggregating at multiple levels dramatically reduces DB write load.

```text
Problem: YouTube video gets 1M views/minute
         → 1M writes/minute to view_count column
         → massive lock contention on one row

Solution: hierarchical aggregation

Level 1: In-memory counter (per app instance, flush every 5s)
           App instance 1: saw 500 views → write "+500" every 5s
           App instance 2: saw 300 views → write "+300" every 5s

Level 2: Redis counter (aggregate across instances, flush every 60s)
           Redis: video:123:views = 800 → write "+800" to DB every 60s

Level 3: DB (source of truth, updated infrequently)
           DB: videos.view_count += 800  (one write per minute, not 1M)
```

**Implementation:**

```java
@Component
public class ViewCountAggregator {

    // Level 1: in-memory (per JVM instance)
    private final ConcurrentHashMap<String, LongAdder> localCounters = new ConcurrentHashMap<>();
    private final StringRedisTemplate redisTemplate;

    public void recordView(String videoId) {
        localCounters.computeIfAbsent(videoId, k -> new LongAdder()).increment();
        // returns immediately — no I/O
    }

    // Level 1 → Level 2: flush local counters to Redis every 5 seconds
    @Scheduled(fixedDelay = 5000)
    public void flushToRedis() {
        localCounters.forEach((videoId, adder) -> {
            long delta = adder.sumThenReset();
            if (delta > 0) {
                redisTemplate.opsForValue().increment("views:" + videoId, delta);
            }
        });
    }
}

@Component
public class ViewCountPersister {

    private final StringRedisTemplate redisTemplate;
    private final VideoRepository videoRepo;

    // Level 2 → Level 3: flush Redis counters to DB every 60 seconds
    @Scheduled(fixedDelay = 60_000)
    @Transactional
    public void flushToDb() {
        Set<String> keys = redisTemplate.keys("views:*");
        if (keys == null || keys.isEmpty()) return;

        keys.forEach(key -> {
            String videoId = key.replace("views:", "");
            String value = redisTemplate.opsForValue().getAndDelete(key);
            if (value != null && Long.parseLong(value) > 0) {
                videoRepo.incrementViewCount(videoId, Long.parseLong(value));
            }
        });
    }
}

// DB update — atomic increment
@Repository
public interface VideoRepository extends JpaRepository<Video, String> {

    @Modifying
    @Query("UPDATE Video v SET v.viewCount = v.viewCount + :delta WHERE v.id = :videoId")
    void incrementViewCount(@Param("videoId") String videoId, @Param("delta") long delta);
}
```

**Write reduction achieved:**

```text
1M views/minute
  → Level 1 flush (every 5s): 12 flushes/minute × N app instances → Redis
  → Level 2 flush (every 60s): 1 flush/minute → DB
  → DB sees: 1 write/minute instead of 1M writes/minute
```

---

## Deep Dives

### Resharding Without Downtime

**Problem:** You have 4 shards. Traffic doubles. You need 8 shards. How do you migrate without downtime?

**Naive approach (causes downtime):**

```text
1. Stop all writes
2. Copy data from 4 shards to 8 shards
3. Update shard routing config
4. Resume writes
→ downtime during step 1-4
```

**Zero-downtime resharding:**

```text
Phase 1: Double-write
  Deploy new code that writes to both old (4) and new (8) shard config
  New writes → both configs simultaneously

Phase 2: Backfill
  Background job copies existing data from old shards to new shards
  Handles deduplication (new writes may have already written to new shards)

Phase 3: Verify
  Compare data between old and new shards
  Run validation queries to confirm consistency

Phase 4: Switch reads
  Update routing to read from new shards (writes still go to both)

Phase 5: Stop double-write
  Update routing to write to new shards only
  Decommission old shards
```

```java
@Service
public class DoubleWriteAdClickService {

    private final ShardDataSourceConfig oldSharding; // 4 shards
    private final ShardDataSourceConfig newSharding; // 8 shards
    private final FeatureFlagService featureFlags;

    public void recordClick(String adId, String userId) {
        AdClick click = new AdClick(adId, userId, Instant.now());

        // always write to old shards
        writeToShard(oldSharding.getShardForKey(adId), click);

        // write to new shards only when double-write flag is enabled
        if (featureFlags.isEnabled("double-write-resharding")) {
            writeToShard(newSharding.getShardForKey(adId), click);
        }
    }
}
```

**With consistent hashing:** Adding nodes only remaps ~1/N of keys — most data stays put.

```text
4 shards → 5 shards with consistent hashing:
  Only 1/5 of keys need to move (vs 50% with simple modulo hashing)
```

---

### Hot Key Problem

**Problem:** One adId receives 10M clicks/second — exceeds what even a single shard can handle.

#### Option 1: Split All Keys (Scatter-Gather)

Append a random suffix to the key, write to multiple sub-keys, aggregate on read.

```java
@Service
public class AdClickService {

    private static final int SPLIT_FACTOR = 100;

    public void recordClick(String adId, String userId) {
        int bucket = ThreadLocalRandom.current().nextInt(SPLIT_FACTOR);
        String shardedKey = adId + ":" + bucket; // "ad123:47"

        DataSource shard = shardConfig.getShardForKey(shardedKey);
        writeToShard(shard, shardedKey, userId);
    }

    // On read: aggregate across all 100 buckets
    public long getTotalClicks(String adId) {
        return IntStream.range(0, SPLIT_FACTOR)
                .parallel()
                .mapToLong(i -> {
                    String key = adId + ":" + i;
                    return clickRepo.countByAdId(key);
                })
                .sum();
    }
}
```

**Tradeoff:** Reads are now scatter-gather (100 queries). Good if reads are infrequent, writes are extreme.

---

#### Option 2: Split Hot Keys Dynamically

Detect hot keys at runtime and split only those.

```java
@Component
public class HotKeyDetector {

    private final ConcurrentHashMap<String, LongAdder> writeRate = new ConcurrentHashMap<>();
    private final Set<String> hotKeys = ConcurrentHashMap.newKeySet();
    private static final long HOT_THRESHOLD = 100_000; // writes/sec

    public void record(String key) {
        writeRate.computeIfAbsent(key, k -> new LongAdder()).increment();
    }

    @Scheduled(fixedDelay = 1000)
    public void detectHotKeys() {
        writeRate.forEach((key, counter) -> {
            long rate = counter.sumThenReset();
            if (rate > HOT_THRESHOLD) {
                hotKeys.add(key);
                log.warn("Hot key detected: {} at {} writes/sec", key, rate);
            }
        });
    }

    public boolean isHot(String key) {
        return hotKeys.contains(key);
    }
}

@Service
public class AdClickService {

    private final HotKeyDetector hotKeyDetector;

    public void recordClick(String adId, String userId) {
        hotKeyDetector.record(adId);

        if (hotKeyDetector.isHot(adId)) {
            // split hot key across buckets
            int bucket = ThreadLocalRandom.current().nextInt(50);
            writeToShard(shardConfig.getShardForKey(adId + ":" + bucket), adId, userId);
        } else {
            writeToShard(shardConfig.getShardForKey(adId), adId, userId);
        }
    }
}
```

---

## Real-World Examples

### Ad Click Aggregator

```text
Challenge: 10M clicks/second during major ad campaign

Write pipeline:
  Click event → Kafka topic (partitioned by adId)
              │
              ▼
  Consumer group (100 instances) → batch writes to Cassandra
              │
              ▼
  Hierarchical aggregation:
    L1: in-memory per consumer (LongAdder, flush every 5s)
    L2: Redis (aggregate, flush every 60s)
    L3: Cassandra (source of truth, updated per minute)

Hot key handling:
  Detected hot adId → split writes across 100 Cassandra partitions
  Read aggregation: scatter-gather across 100 partitions → sum

Result: 10M clicks/sec → 10K DB writes/sec (1000x reduction)
```

---

### Metrics Monitoring (Prometheus-like)

```text
Challenge: 100K time-series data points/second from IoT fleet

Write pipeline:
  Sensor data → Kafka (partitioned by metricId + hostname)
              │
              ▼
  Consumer → batch insert to TimescaleDB (hypertable, partitioned by time)
              │
              ▼
  TimescaleDB continuous aggregates:
    Raw data (1s resolution) → auto-aggregate to 1m, 1h, 1d
    Dashboard queries hit aggregates, not raw data

Retention:
  Raw (1s):  keep 7 days
  1m agg:    keep 30 days
  1h agg:    keep 1 year
  1d agg:    keep forever

Result: 100K writes/sec in, dashboards query pre-aggregated data (fast reads)
```

---

### Strava Segment Leaderboard

```text
Challenge: 50K athletes finish the same popular cycling segment simultaneously

Naive: each finish → UPDATE leaderboard WHERE rank < user_time → massive contention

Write pipeline:
  Segment finish → Kafka topic "segment-completions"
               │
               ▼
  Consumer → Redis Sorted Set (ZADD segment:leaderboard userId time)
               │  (Redis handles 100K writes/sec, atomic, no contention)
               ▼
  Background job → persist top-N to DB every minute
               │
               ▼
  DB leaderboard table (source of truth, updated infrequently)

Why Redis Sorted Set works:
  ZADD is O(log N) — atomic, no lock contention
  ZRANGE for reads — sorted, paginated
  Handles 100K concurrent ZADDs without contention
```

---

## Interview Cheat Sheet

### When to Use Which Strategy

| Scenario | Strategy |
|----------|---------|
| Write throughput < 50K/sec | Vertical scaling + index tuning |
| Need append-only high throughput | Cassandra / LSM-tree DB |
| Write throughput > 50K/sec, shardable key | Horizontal sharding |
| Large rows with hot/cold columns | Vertical partitioning |
| Bursty write traffic | Kafka as write buffer |
| Unacceptable writes during spikes | Load shedding + rate limiting |
| High-write counters (views, likes) | Hierarchical aggregation (RAM → Redis → DB) |
| Bulk data ingest | Batch inserts (Kafka consumer → saveAll) |
| Extreme single-key contention | Key splitting (scatter-gather) |

---

### Write Scaling Comparison

| Strategy | Write Throughput Gain | Complexity | Consistency |
|----------|-----------------------|------------|-------------|
| Vertical scaling | 2–5x | None | Strong |
| LSM-tree DB (Cassandra) | 10–100x | Low | Eventual |
| Read replicas | 0 (writes unaffected) | Low | Strong |
| Horizontal sharding | N × base throughput | High | Strong per shard |
| Kafka buffer | Absorbs any burst | Medium | Eventual |
| Batching | 10–100x fewer DB calls | Low | Slight delay |
| Hierarchical aggregation | 100–1000x | Medium | Eventual |
| Key splitting | N × per-key throughput | Medium | Scatter-gather reads |

---

### Common Deep Dive Answers

**"How do you handle resharding when you need to add more shards?"**

- Use consistent hashing to minimize data movement (1/N keys move vs 50% with modulo)
- Double-write phase: write to old and new shards simultaneously
- Backfill old data to new shards in background
- Validate, then switch reads to new shards, then stop double-write

**"What happens when a single key is too hot for even one shard?"**

- Detect hot keys dynamically using write rate counters
- Split writes across N sub-keys (adId:0, adId:1, ... adId:N)
- Aggregate sub-keys at read time (scatter-gather)
- Use Redis atomic operations for counters that don't need sharding

**"How do you ensure no data loss when using queues?"**

- Kafka is durable — messages persist until consumer commits offset
- Consumer commits offset only after successful DB write
- At-least-once delivery — make consumers idempotent (upsert, not insert)

**"How do you handle write-heavy counters without DB contention?"**

- In-memory `LongAdder` per JVM instance (nanosecond increments)
- Periodic flush to Redis (microsecond increments, atomic)
- Periodic flush from Redis to DB (one write per flush interval)
- Accept eventual consistency — view counts don't need to be real-time exact

---

### Architecture Decision Flowchart

```text
Write throughput is the bottleneck?
          │
          ▼
Have you tuned indexes and used batch inserts?
   NO → Do that first (free wins)
   YES → continue
          │
          ▼
Is your write pattern append-only (no updates)?
   YES → Consider Cassandra / LSM-tree DB
   NO  → continue
          │
          ▼
Can writes be sharded by a natural key (userId, adId)?
   YES → Horizontal sharding
   NO  → continue
          │
          ▼
Is the write traffic bursty?
   YES → Kafka write buffer (absorbs bursts)
   NO  → continue
          │
          ▼
Are you writing to a single counter / aggregated value?
   YES → Hierarchical aggregation (RAM → Redis → DB)
   NO  → continue
          │
          ▼
Is one key receiving extreme traffic?
   YES → Key splitting (scatter-gather on reads)
```

---

### Key Terms Quick Reference

| Term | Meaning |
|------|---------|
| **LSM-tree** | Log-structured merge tree — sequential writes, used by Cassandra, RocksDB |
| **WAL** | Write-Ahead Log — append before updating data, enables crash recovery |
| **Horizontal sharding** | Split rows across DB instances by shard key |
| **Vertical partitioning** | Split columns across tables — separate hot from cold |
| **Write buffer** | Queue (Kafka) that absorbs write bursts before DB |
| **Load shedding** | Intentionally dropping writes under extreme load |
| **Hierarchical aggregation** | Multi-level aggregation: RAM → Redis → DB |
| **Key splitting** | Distribute one hot key across N sub-keys to spread load |
| **Consistent hashing** | Shard routing that minimizes data movement when nodes change |
| **Double-write** | Temporarily writing to both old and new shards during resharding |
| **Idempotent write** | Write that can be safely retried without side effects |
| **Token bucket** | Rate limiting algorithm that allows controlled bursts |

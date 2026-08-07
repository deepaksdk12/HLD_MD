# Numbers to Know for System Design Interviews

Modern hardware capabilities change constantly. Using outdated numbers in an interview signals book knowledge over hands-on experience. The decisions around when to shard, whether to cache, how to handle large objects — all depend on an accurate sense of what today's hardware can actually handle.

---

## Modern Hardware Limits (2026)

### Compute

| Instance Type | vCPUs | RAM | Use Case |
|---|---|---|---|
| AWS m6i.32xlarge | 128 | 512 GiB | General workloads |
| AWS x1e.32xlarge | 128 | 4 TB | In-memory databases |
| AWS u-24tb1.metal | 448 | 24 TB | SAP HANA, large in-memory |
| AWS c6i.32xlarge | 128 | 256 GiB | Compute-intensive |

**Key insight:** Many systems that once required distributed architecture can run on a single machine today.

### Storage

| Type | Capacity | Throughput | Latency |
|---|---|---|---|
| NVMe SSD (local) | Up to 60 TB (i3en.24xlarge) | 16 GB/s | ~100 µs |
| Network SSD (EBS gp3) | Up to 64 TB per volume | 1 GB/s | ~1 ms |
| HDD (d3en.12xlarge) | 336 TB | 900 MB/s | ~5 ms |
| Object Storage (S3) | Effectively unlimited | Multi-GB/s | 10–50 ms |

### Network

| Scope | Bandwidth | Latency |
|---|---|---|
| Within same AZ | 25–100 Gbps | < 1 ms |
| Cross-AZ (same region) | Limited by instance NIC | 1–2 ms |
| Cross-region | Limited by instance NIC | 50–150 ms |
| Cross-continent | — | 100–300 ms |

---

## Applying Numbers: Component by Component

### Caching (Redis / Memcached)

| Metric | Number |
|---|---|
| Single Redis node RAM | Up to 500 GB |
| Redis throughput | ~1 million ops/sec |
| Redis read latency | < 1 ms (sub-millisecond) |
| Typical cache hit ratio target | 90–99% |
| Cache entry overhead | ~70 bytes per key (Redis) |

**Practical sizing:**

```
10 million cached objects × 1 KB each = 10 GB  → fits on a single Redis node
```

**What this means in interviews:**

- Don't reach for Redis Cluster until you exceed ~500 GB of data or ~1M ops/sec
- A single Redis instance handles most mid-scale systems
- Cache-aside is the most common pattern; write-through for strong consistency

---

### Databases (PostgreSQL / MySQL)

| Metric | Number |
|---|---|
| Single PostgreSQL instance (RDS r6g.16xlarge) | 512 GB RAM, 64 vCPUs |
| Practical DB size before sharding | 5–10 TB (with read replicas) |
| Read replicas per primary | Up to 5–15 (varies by DB) |
| Rows scanned per sec (full scan) | ~100M rows/sec on SSD |
| Indexed lookup | < 5 ms p99 |
| Connection pool limit (RDS) | ~5,000 connections |
| Write throughput (single primary) | ~10,000–50,000 writes/sec |

**What this means in interviews:**

- Don't shard at 100 GB — that was a 2015 constraint. Shard at 5–10 TB+
- Read replicas handle 5–10x read scale before you need sharding
- Vertical scaling first: RDS r6g.16xlarge handles most production workloads
- PgBouncer / connection pooling is needed before you hit connection limits, not DB size limits

**When to actually shard:**

```
Users: 1B → avg profile: 2 KB → total = 2 TB → single DB, fine
Posts: 10B → avg post: 5 KB → total = 50 TB → shard by user_id
```

---

### Application Servers

| Metric | Number |
|---|---|
| Requests/sec per server (simple API) | 10,000–50,000 RPS |
| Requests/sec (CPU-bound, ML inference) | 100–500 RPS |
| Memory per Java/Node process | 256 MB – 4 GB |
| Thread pool size (Java) | 100–500 threads |
| P99 latency target (APIs) | < 100 ms |
| HTTP keep-alive connections | Thousands per server |

**What this means in interviews:**

- A single app server handles 10K–50K RPS for typical CRUD workloads
- 1M RPS needs ~20–100 servers behind a load balancer — not thousands
- Stateless servers scale horizontally; session state goes in Redis, not in-process

---

### Message Queues (Kafka)

| Metric | Number |
|---|---|
| Kafka throughput per broker | 100 MB/s–1 GB/s write |
| Kafka messages/sec per broker | 1–2 million msg/sec |
| Message retention | Days to weeks (disk-based) |
| Consumer lag recovery | Can replay from beginning |
| Partition count per topic | Hundreds to thousands |
| Replication factor | 3 (standard) |

**Kafka vs alternatives:**

| Queue | Throughput | Ordering | Retention | Use Case |
|---|---|---|---|---|
| Kafka | Very high | Per-partition | Days–weeks | Event streaming, audit log |
| RabbitMQ | Moderate | Per-queue | Until consumed | Task queues, RPC |
| SQS | High | Best-effort | 14 days max | AWS-native async |
| Redis Streams | High | Per-stream | Configurable | Low-latency event bus |

---

## Latency Cheat Sheet

Knowing these by heart lets you do back-of-envelope math in real-time.

| Operation | Latency |
|---|---|
| L1 cache reference | 0.5 ns |
| L2 cache reference | 7 ns |
| RAM access | 100 ns |
| SSD random read (4KB) | 100 µs (0.1 ms) |
| HDD random read | 10 ms |
| Redis get (same AZ) | 0.5–1 ms |
| DB indexed query (same AZ) | 1–5 ms |
| DB full table scan (1M rows) | 100 ms – 1 s |
| Network: same AZ | < 1 ms |
| Network: cross-AZ | 1–2 ms |
| Network: cross-region (US) | 50–100 ms |
| Network: cross-continent | 150–300 ms |

**Rule of thumb:** Memory is ~100x faster than SSD. SSD is ~100x faster than HDD.

---

## Throughput Cheat Sheet

| Component | Throughput |
|---|---|
| Single app server (CRUD API) | 10K–50K RPS |
| Single PostgreSQL primary | 10K–50K writes/sec |
| Single Redis node | ~1M ops/sec |
| Single Kafka broker | 1–2M msgs/sec |
| CDN (CloudFront) | Tbps aggregate |
| Load balancer (ALB) | Millions of RPS |

---

## Back-of-Envelope: Template

```
Daily active users (DAU):          X million
Requests per user per day:         Y
Total requests/day:                X × Y million
Requests/sec (RPS):                total / 86,400
Peak RPS (3× average):             RPS × 3

Storage per record:                Z KB
New records/day:                   N million
Storage/day:                       N × Z GB
Storage/year:                      × 365
```

**Example — Twitter-like system:**

```
DAU: 300M users
Posts/user/day: 2
Total posts/day: 600M
Posts/sec: 600M / 86,400 ≈ 7,000 writes/sec  →  single DB primary, fine

Post size: 1 KB text + 500 KB media avg
Storage/day: 600M × 500 KB = 300 TB/day (media)  →  object storage (S3)
             600M × 1 KB   = 600 GB/day (text)   →  DB, partitioned by date
```

---

## Common Interview Mistakes

### 1. Premature Sharding

**Wrong:** "We have 10 million users, so we need to shard the database."

**Right:** 10M users × 2 KB = 20 GB — trivially fits on a single PostgreSQL instance with room to grow 500x before sharding is needed.

Shard when:
- Data exceeds 5–10 TB per table
- Write throughput exceeds ~50K writes/sec on a single primary
- Single-region latency requirements force geo-partitioning

### 2. Overestimating Latency

**Wrong:** "Database calls take 100ms, so caching everything is critical."

**Right:** An indexed PostgreSQL query in the same AZ takes 1–5 ms. Cache when:
- You need sub-millisecond response
- The same data is read thousands of times (read amplification)
- Computation is expensive (not just lookup)

### 3. Over-engineering Write Throughput

**Wrong:** "We need Kafka because we have 1,000 writes/sec."

**Right:** PostgreSQL handles 10K–50K writes/sec. Kafka is warranted when:
- You need fan-out to multiple consumers
- You need event replay / audit log
- Writes are bursty and consumers need decoupling
- Throughput genuinely exceeds what the DB can absorb

### 4. Ignoring Vertical Scaling

Distributed systems add complexity — coordination, partial failures, consistency problems. The modern default should be:

```
1. Vertical scale first (bigger machine)
2. Read replicas for read-heavy workloads
3. Caching for repeated reads
4. Horizontal scale / sharding only when the above are exhausted
```

### 5. Forgetting About Connections

The bottleneck before DB size is often **connection count**. Each connection consumes ~10 MB RAM. At 500 app servers × 10 connections each = 5,000 connections — already at RDS limits.

Fix: connection pooler (PgBouncer) sits between app and DB, multiplexing thousands of app connections into tens of DB connections.

---

## Cost Awareness

Interviewers at senior levels sometimes probe cost trade-offs:

| Resource | Rough AWS Cost |
|---|---|
| RDS r6g.4xlarge (128 GB RAM) | ~$700/month |
| EC2 m6i.4xlarge (64 GB RAM) | ~$550/month |
| S3 storage | $23/TB/month |
| S3 GET requests | $0.40 per million |
| Data transfer out (internet) | $90/TB |
| ElastiCache r6g.4xlarge | ~$600/month |
| Kafka (MSK, 3 brokers) | ~$1,500/month |

**Key cost insight for interviews:** Storage is cheap ($23/TB/month on S3). Compute is moderate. **Data egress is expensive** — design to minimize cross-region data transfer and keep hot data close to compute.

---

## Interview Quick Reference

| Question | Modern Answer |
|---|---|
| When to shard? | Data > 5–10 TB or writes > 50K/sec |
| When to cache? | Same data read 1000s of times, or sub-ms needed |
| When to add read replicas? | Read:write ratio > 5:1 |
| When to use Kafka? | Fan-out, replay, or burst decoupling |
| Max single Redis node? | ~500 GB RAM, ~1M ops/sec |
| Max single Postgres? | ~5–10 TB practical, ~50K writes/sec |
| Typical API server RPS? | 10K–50K RPS per instance |
| Same-AZ latency target? | < 5 ms for DB, < 1 ms for cache |

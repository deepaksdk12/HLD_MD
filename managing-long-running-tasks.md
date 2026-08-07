# Managing Long-Running Tasks — System Design Pattern

## Table of Contents

1. [The Problem](#the-problem)
2. [The Solution — Async Job Pattern](#the-solution--async-job-pattern)
3. [Trade-offs](#trade-offs)
4. [How to Implement](#how-to-implement)
   - [Message Queue](#message-queue)
   - [Workers](#workers)
   - [Job Status Tracking](#job-status-tracking)
   - [Putting It Together](#putting-it-together)
5. [Deep Dives](#deep-dives)
   - [Handling Failures](#handling-failures)
   - [Handling Repeated Failures](#handling-repeated-failures)
   - [Preventing Duplicate Work](#preventing-duplicate-work)
   - [Managing Queue Backpressure](#managing-queue-backpressure)
   - [Handling Mixed Workloads](#handling-mixed-workloads)
   - [Orchestrating Job Dependencies](#orchestrating-job-dependencies)
6. [Real-World Examples](#real-world-examples)
7. [Interview Cheat Sheet](#interview-cheat-sheet)

---

## The Problem

Some operations take too long to run synchronously in a web request.

```text
Synchronous (bad for long tasks):

User clicks "Generate PDF Report"
       │
       ▼
Web Server receives request
       │ (45 seconds pass...)
       │ CPU grinding, memory held, user staring at spinner
       ▼
Response sent after 45s

Problems:
  - Load balancer timeout: typically 30–60s → request killed before completion
  - Web server thread held for 45s → can't serve other users
  - User has no progress feedback
  - Retry on failure = re-run entire 45s job
  - Server crash mid-job → work lost, user never notified
```

**Examples of tasks that break synchronous handling:**

| Task | Duration | Why Async |
|------|----------|-----------|
| Video transcoding | Minutes | Exceeds any HTTP timeout |
| PDF report generation | 10–60s | Near or exceeds load balancer timeout |
| Bulk email newsletter | Minutes | Thousands of API calls |
| CSV import (millions of rows) | Minutes | I/O bound, blocks threads |
| ML model inference (batch) | Seconds–minutes | CPU/GPU intensive |
| Image resize + thumbnail | 1–5s | Fine synchronously if < 5s, async if many sizes |

---

## The Solution — Async Job Pattern

Split the request into two phases:

```text
Phase 1: Acknowledge (milliseconds)

User clicks "Generate PDF"
       │
       ▼
Web Server:
  1. Validate request (auth, params)
  2. Create job record in DB (status = PENDING)
  3. Enqueue job ID to message queue
  4. Return job ID to client immediately
       │
       ▼
Client receives: { "jobId": "abc-123", "status": "PENDING" }
→ User sees: "Your report is being generated..."

Phase 2: Process (background, no timeout)

Worker process:
  1. Poll queue → dequeue job ID "abc-123"
  2. Fetch job details from DB
  3. Execute actual work (generate PDF)
  4. Update job status in DB: COMPLETED + result URL
  5. Notify user (optional: WebSocket, SSE, email, push)
       │
       ▼
Client polls GET /jobs/abc-123 → receives COMPLETED + download URL
```

---

## Trade-offs

### What You Gain

- **No timeouts** — workers run without HTTP timeout constraints
- **User feedback** — client can poll for progress (0% → 50% → 100%)
- **Resilience** — job survives server restarts (persisted in DB + queue)
- **Scalability** — add more workers independently of web servers
- **Retry safety** — failed jobs retry without re-submitting from client
- **Decoupling** — web servers and workers scale independently

### What You Lose

- **Simplicity** — two systems (web server + worker) instead of one
- **Immediate result** — user can't get result in the same HTTP response
- **Consistency** — job state is eventually consistent (DB update after processing)
- **Observability** — harder to trace a request across web server → queue → worker

---

## How to Implement

### Message Queue

The queue decouples job submission (web server) from job execution (workers).

**Queue options:**

| Queue | Best For | Durability | Ordering |
|-------|---------|------------|---------|
| Kafka | High-throughput, replay needed | Yes | Per partition |
| RabbitMQ | Flexible routing, priority queues | Yes | FIFO per queue |
| Redis (List/Stream) | Simple, low latency | Limited | FIFO |
| AWS SQS | AWS-native, managed | Yes | Best-effort |
| AWS SQS FIFO | Ordered, exactly-once | Yes | Strict FIFO |
| Google Pub/Sub | GCP-native, fan-out | Yes | Best-effort |

**Spring Boot with Kafka (recommended for high scale):**

```java
// Job event published by web server
@Service
public class JobSubmissionService {

    private final KafkaTemplate<String, JobEvent> kafkaTemplate;
    private final JobRepository jobRepo;

    public JobSubmissionResponse submit(JobRequest req, String userId) {
        // Create and persist job record
        Job job = Job.builder()
                .id(UUID.randomUUID().toString())
                .type(req.getJobType())
                .userId(userId)
                .payload(req.getPayload())
                .status(JobStatus.PENDING)
                .createdAt(Instant.now())
                .build();
        jobRepo.save(job);

        // Enqueue job ID (not the full payload — payload is in DB)
        JobEvent event = new JobEvent(job.getId(), job.getType());
        kafkaTemplate.send("jobs", job.getId(), event); // jobId as key → same partition

        return JobSubmissionResponse.builder()
                .jobId(job.getId())
                .status(JobStatus.PENDING)
                .pollUrl("/api/jobs/" + job.getId())
                .build();
    }
}
```

**Spring Boot with SQS (AWS):**

```java
@Service
public class JobSubmissionService {

    private final SqsTemplate sqsTemplate;
    private final JobRepository jobRepo;

    public JobSubmissionResponse submit(JobRequest req, String userId) {
        Job job = createAndPersistJob(req, userId);

        sqsTemplate.send("jobs-queue.fifo",
                SqsMessageAttributes.builder()
                        .messageGroupId(job.getType())         // FIFO grouping
                        .messageDeduplicationId(job.getId())   // dedup key
                        .build(),
                new JobEvent(job.getId(), job.getType()));

        return JobSubmissionResponse.builder()
                .jobId(job.getId())
                .status(JobStatus.PENDING)
                .build();
    }
}
```

---

### Workers

Workers are independent processes that consume jobs from the queue and execute them.

```java
@Component
public class JobWorker {

    private final JobRepository jobRepo;
    private final Map<String, JobHandler> handlers;

    // Kafka consumer
    @KafkaListener(topics = "jobs", groupId = "job-workers", concurrency = "5")
    public void processJob(ConsumerRecord<String, JobEvent> record) {
        String jobId = record.value().getJobId();

        Job job = jobRepo.findById(jobId)
                .orElseThrow(() -> new JobNotFoundException(jobId));

        try {
            // Mark as running
            job.setStatus(JobStatus.RUNNING);
            job.setStartedAt(Instant.now());
            job.setWorkerId(getWorkerId());
            jobRepo.save(job);

            // Dispatch to appropriate handler
            JobHandler handler = handlers.get(job.getType());
            JobResult result = handler.execute(job);

            // Mark as completed
            job.setStatus(JobStatus.COMPLETED);
            job.setCompletedAt(Instant.now());
            job.setResultUrl(result.getOutputUrl());
            job.setProgress(100);
            jobRepo.save(job);

        } catch (Exception e) {
            job.setStatus(JobStatus.FAILED);
            job.setErrorMessage(e.getMessage());
            job.setAttempts(job.getAttempts() + 1);
            jobRepo.save(job);

            // Re-throw to trigger Kafka retry/DLQ behavior
            throw new JobProcessingException("Job failed: " + jobId, e);
        }
    }
}

// Handler for PDF generation
@Component("PDF_REPORT")
public class PdfReportHandler implements JobHandler {

    @Override
    public JobResult execute(Job job) {
        PdfReportPayload payload = parse(job.getPayload());

        // Update progress as work proceeds
        updateProgress(job.getId(), 10, "Fetching data...");
        List<ActivityData> data = activityRepo.fetchUserActivity(payload.getUserId());

        updateProgress(job.getId(), 40, "Aggregating...");
        ReportData report = aggregationService.aggregate(data);

        updateProgress(job.getId(), 70, "Rendering PDF...");
        byte[] pdfBytes = pdfRenderer.render(report);

        updateProgress(job.getId(), 90, "Uploading...");
        String s3Key = s3Service.upload(pdfBytes, "reports/" + job.getId() + ".pdf");

        return JobResult.builder()
                .outputUrl(s3Service.generatePresignedUrl(s3Key))
                .build();
    }
}
```

---

### Job Status Tracking

Clients poll the status endpoint to check progress.

**Job DB schema:**

```java
@Entity
@Table(name = "jobs")
public class Job {

    @Id
    private String id;

    @Enumerated(EnumType.STRING)
    private JobStatus status; // PENDING, RUNNING, COMPLETED, FAILED, CANCELLED

    private String type;       // "PDF_REPORT", "VIDEO_TRANSCODE", etc.
    private String userId;

    @Column(columnDefinition = "jsonb")
    private String payload;    // job-specific input params

    private String resultUrl;  // populated on COMPLETED
    private String errorMessage; // populated on FAILED
    private int progress;      // 0–100
    private String progressMessage; // "Rendering page 3 of 10..."

    private int attempts;      // retry counter
    private int maxAttempts;   // configured max retries

    private String workerId;   // which worker picked this up
    private Instant createdAt;
    private Instant startedAt;
    private Instant completedAt;
    private Instant expiresAt; // auto-cleanup after N days
}
```

**Status polling endpoint:**

```java
@RestController
@RequestMapping("/api/jobs")
public class JobController {

    @GetMapping("/{jobId}")
    public ResponseEntity<JobStatusResponse> getJobStatus(@PathVariable String jobId,
                                                           @AuthenticationPrincipal User user) {
        Job job = jobRepo.findById(jobId)
                .orElseThrow(() -> new NotFoundException(jobId));

        if (!job.getUserId().equals(user.getId())) {
            throw new ForbiddenException("Access denied");
        }

        return ResponseEntity.ok(JobStatusResponse.builder()
                .jobId(job.getId())
                .status(job.getStatus())
                .progress(job.getProgress())
                .progressMessage(job.getProgressMessage())
                .resultUrl(job.getResultUrl())           // null until COMPLETED
                .errorMessage(job.getErrorMessage())     // null until FAILED
                .createdAt(job.getCreatedAt())
                .completedAt(job.getCompletedAt())
                .build());
    }
}
```

**Push notification on completion (SSE alternative to polling):**

```java
// Worker notifies client via SSE when job completes
@Component
public class JobCompletionNotifier {

    private final SseController sseController;

    public void notifyComplete(Job job) {
        sseController.pushToUser(job.getUserId(), SseEvent.builder()
                .type("job-complete")
                .data(Map.of(
                    "jobId", job.getId(),
                    "status", "COMPLETED",
                    "resultUrl", job.getResultUrl()
                ))
                .build());
    }
}
```

---

### Putting It Together

```text
Full flow for "Generate PDF Report":

1. POST /api/reports/generate
   └─ Web Server: validate, create job (PENDING), enqueue jobId → Kafka
   └─ Response: { jobId: "abc-123", pollUrl: "/api/jobs/abc-123" }

2. Client polls GET /api/jobs/abc-123 every 2 seconds
   └─ Response: { status: "PENDING", progress: 0 }

3. Worker dequeues "abc-123" from Kafka
   └─ Updates DB: status = RUNNING, progress = 0

4. Worker executes PdfReportHandler
   └─ Progress updates: 10% → 40% → 70% → 90%

5. Client polls GET /api/jobs/abc-123
   └─ Response: { status: "RUNNING", progress: 70, message: "Rendering PDF..." }

6. Worker completes, uploads PDF to S3
   └─ Updates DB: status = COMPLETED, resultUrl = "https://..."

7. Client polls GET /api/jobs/abc-123
   └─ Response: { status: "COMPLETED", resultUrl: "https://..." }
   └─ Client: download PDF
```

---

## Deep Dives

### Handling Failures

**Transient failures** (network blip, DB timeout) — retry automatically:

```java
@KafkaListener(topics = "jobs", groupId = "job-workers")
@RetryableTopic(
    attempts = "3",
    backoff = @Backoff(delay = 1000, multiplier = 2), // 1s, 2s, 4s
    dltTopicSuffix = "-dlt"  // failed after 3 retries → dead letter topic
)
public void processJob(ConsumerRecord<String, JobEvent> record) {
    // ... execute job
}

// Dead letter topic consumer — alert and inspect failed jobs
@KafkaListener(topics = "jobs-dlt")
public void handleDeadLetter(ConsumerRecord<String, JobEvent> record) {
    String jobId = record.value().getJobId();
    Job job = jobRepo.findById(jobId).orElseThrow();
    job.setStatus(JobStatus.FAILED);
    job.setErrorMessage("Exceeded max retry attempts");
    jobRepo.save(job);

    alertService.notifyJobFailed(job);
}
```

**With SQS — visibility timeout as implicit lock:**

```java
@SqsListener(value = "jobs-queue", deletionPolicy = SqsMessageDeletionPolicy.NEVER)
public void processJob(JobEvent event, Acknowledgment ack) {
    Job job = jobRepo.findById(event.getJobId()).orElseThrow();

    try {
        executeJob(job);
        ack.acknowledge(); // deletes message from SQS only on success
    } catch (Exception e) {
        // Do NOT acknowledge — SQS will re-deliver after visibility timeout (e.g., 5 min)
        // After maxReceiveCount exceeded → SQS moves to DLQ automatically
        log.error("Job failed, will be retried: {}", event.getJobId(), e);
    }
}
```

---

### Handling Repeated Failures

After N retries, the job is permanently failed. Strategies:

**1. Dead Letter Queue (DLQ):**

```text
Job fails 3 times → moved to DLQ → alert oncall → manual inspection/replay
```

**2. Exponential backoff with jitter:**

```java
public Duration getRetryDelay(int attemptNumber) {
    long baseDelayMs = 1000;
    long maxDelayMs = 30_000;
    long delay = Math.min(baseDelayMs * (1L << attemptNumber), maxDelayMs);
    long jitter = ThreadLocalRandom.current().nextLong(delay / 2);
    return Duration.ofMillis(delay + jitter); // e.g. 1s, 2.3s, 4.8s, 9.1s...
}
```

**3. Partial retry (resume from checkpoint):**

```java
// For jobs with multiple steps, checkpoint after each step
@Transactional
public void executeVideoTranscode(Job job) {
    VideoTranscodePayload payload = parse(job.getPayload());

    // Check if step already completed (idempotent resume)
    if (!job.isStepCompleted("download")) {
        downloadFromS3(payload.getSourceKey());
        job.markStepCompleted("download");
        jobRepo.save(job);
    }

    if (!job.isStepCompleted("transcode-360p")) {
        transcode(payload.getSourceKey(), "360p");
        job.markStepCompleted("transcode-360p");
        jobRepo.save(job);
    }

    if (!job.isStepCompleted("transcode-1080p")) {
        transcode(payload.getSourceKey(), "1080p");
        job.markStepCompleted("transcode-1080p");
        jobRepo.save(job);
    }

    // Job restarts from last completed step, not from scratch
}
```

---

### Preventing Duplicate Work

**Problem:** Message delivered twice (Kafka at-least-once) → job runs twice → double charges, duplicate emails.

**Solution: Idempotency check before execution**

```java
@KafkaListener(topics = "jobs")
public void processJob(ConsumerRecord<String, JobEvent> record) {
    String jobId = record.value().getJobId();
    Job job = jobRepo.findById(jobId).orElseThrow();

    // Idempotency check — only process if still PENDING
    // Use optimistic locking to prevent race condition between workers
    int updated = jobRepo.claimJob(jobId, JobStatus.PENDING, JobStatus.RUNNING, getWorkerId());
    if (updated == 0) {
        // Another worker already claimed this job — skip
        log.info("Job {} already claimed, skipping", jobId);
        return;
    }

    executeJob(job);
}

// Atomic claim using conditional UPDATE
@Repository
public interface JobRepository extends JpaRepository<Job, String> {

    @Modifying
    @Query("UPDATE Job j SET j.status = :newStatus, j.workerId = :workerId, j.startedAt = CURRENT_TIMESTAMP " +
           "WHERE j.id = :jobId AND j.status = :expectedStatus")
    int claimJob(@Param("jobId") String jobId,
                 @Param("expectedStatus") JobStatus expectedStatus,
                 @Param("newStatus") JobStatus newStatus,
                 @Param("workerId") String workerId);
}
```

**Idempotency for job submission (prevent duplicate submissions):**

```java
@PostMapping("/reports/generate")
public JobSubmissionResponse generateReport(@RequestBody ReportRequest req,
                                             @AuthenticationPrincipal User user) {
    // Idempotency key from client (prevents double-submit on network retry)
    String idempotencyKey = req.getIdempotencyKey(); // e.g., UUID generated by client

    // Check if job already submitted with this key
    Optional<Job> existing = jobRepo.findByIdempotencyKey(idempotencyKey);
    if (existing.isPresent()) {
        return JobSubmissionResponse.fromExisting(existing.get()); // return same jobId
    }

    return jobSubmissionService.submit(req, user.getId(), idempotencyKey);
}
```

---

### Managing Queue Backpressure

**Problem:** Producers enqueue faster than workers can process → queue grows unbounded → memory/disk exhaustion.

```text
Normal:  Producer 100 jobs/sec → Queue → Workers 100 jobs/sec → Queue depth: stable

Spike:   Producer 1000 jobs/sec → Queue → Workers 100 jobs/sec → Queue grows 900/sec
         After 1 hour: 3.24M jobs in queue → memory exhausted → queue crashes
```

**Solutions:**

**1. Auto-scale workers based on queue depth:**

```java
@Component
public class WorkerAutoScaler {

    private final KafkaAdminClient kafkaAdmin;
    private final WorkerPool workerPool;

    @Scheduled(fixedDelay = 30_000)
    public void adjustWorkerCount() {
        long queueDepth = getQueueDepth("jobs");
        int currentWorkers = workerPool.getActiveCount();

        if (queueDepth > 10_000 && currentWorkers < MAX_WORKERS) {
            workerPool.scaleUp(Math.min(currentWorkers * 2, MAX_WORKERS));
            log.info("Scaled workers up to {}", workerPool.getActiveCount());
        } else if (queueDepth < 100 && currentWorkers > MIN_WORKERS) {
            workerPool.scaleDown(Math.max(currentWorkers / 2, MIN_WORKERS));
        }
    }

    private long getQueueDepth(String topic) {
        // Sum of lag across all partitions for consumer group "job-workers"
        return kafkaAdmin.listConsumerGroupOffsets("job-workers")
                .stream()
                .mapToLong(OffsetAndMetadata::offset)
                .sum();
    }
}
```

**2. Rate limit job submission:**

```java
@PostMapping("/reports/generate")
public JobSubmissionResponse generateReport(@RequestBody ReportRequest req,
                                             @AuthenticationPrincipal User user) {
    // Check queue depth before accepting new jobs
    long queueDepth = queueMonitor.getDepth("jobs");
    if (queueDepth > MAX_QUEUE_DEPTH) {
        throw new ServiceUnavailableException("System busy, please try again later");
    }

    // Per-user rate limit: max 5 concurrent jobs
    long activeJobs = jobRepo.countByUserIdAndStatus(user.getId(), JobStatus.RUNNING);
    if (activeJobs >= 5) {
        throw new TooManyRequestsException("Max concurrent jobs reached");
    }

    return jobSubmissionService.submit(req, user.getId());
}
```

**3. Job TTL and expiry:**

```java
// Jobs that have been pending too long are expired (queue was backed up)
@Scheduled(cron = "0 * * * * *") // every minute
@Transactional
public void expireStaleJobs() {
    Instant cutoff = Instant.now().minus(Duration.ofHours(1));
    int expired = jobRepo.expireStalePendingJobs(cutoff);
    if (expired > 0) {
        log.warn("Expired {} stale pending jobs", expired);
        metrics.increment("jobs.expired", expired);
    }
}
```

---

### Handling Mixed Workloads

**Problem:** Fast 2-second jobs stuck behind slow 45-second jobs → high-priority work is delayed.

**Solution: Priority queues / separate queues per job type**

```text
Separate queues by priority or type:
  "jobs-critical"  → 50 workers (SLA: process within 5s)
  "jobs-standard"  → 20 workers (SLA: process within 60s)
  "jobs-batch"     →  5 workers (SLA: process within 1h)
```

```java
@Service
public class JobSubmissionService {

    public JobSubmissionResponse submit(JobRequest req, String userId) {
        Job job = createJob(req, userId);

        // Route to appropriate queue based on job type and priority
        String queue = determineQueue(job);
        kafkaTemplate.send(queue, job.getId(), new JobEvent(job.getId(), job.getType()));

        return JobSubmissionResponse.from(job);
    }

    private String determineQueue(Job job) {
        return switch (job.getType()) {
            case "PASSWORD_RESET_EMAIL" -> "jobs-critical";   // must be instant
            case "THUMBNAIL_RESIZE"     -> "jobs-standard";   // user is waiting
            case "VIDEO_TRANSCODE"      -> "jobs-standard";   // user is waiting (progress shown)
            case "BULK_EMAIL"           -> "jobs-batch";      // background, no rush
            case "ANALYTICS_REPORT"     -> "jobs-batch";      // background
            default -> "jobs-standard";
        };
    }
}

// Separate listener for each queue with different concurrency
@KafkaListener(topics = "jobs-critical", groupId = "workers-critical", concurrency = "50")
public void processCritical(ConsumerRecord<String, JobEvent> record) { process(record); }

@KafkaListener(topics = "jobs-standard", groupId = "workers-standard", concurrency = "20")
public void processStandard(ConsumerRecord<String, JobEvent> record) { process(record); }

@KafkaListener(topics = "jobs-batch", groupId = "workers-batch", concurrency = "5")
public void processBatch(ConsumerRecord<String, JobEvent> record) { process(record); }
```

---

### Orchestrating Job Dependencies

**Problem:** Job B must run after Job A completes. Job D requires both B and C.

```text
DAG of jobs:
  A (fetch data)
  └── B (aggregate)
      └── D (generate report) ← needs B and C both done
  └── C (fetch images)
```

**Solution: Chain jobs via completion events**

```java
@Component("FETCH_DATA")
public class FetchDataHandler implements JobHandler {

    private final JobSubmissionService jobSubmissionService;

    @Override
    public JobResult execute(Job job) {
        // Do work...
        List<ActivityData> data = activityRepo.fetch(job.getUserId());
        String dataKey = s3.upload(data);

        // On completion, enqueue dependent jobs
        jobSubmissionService.submitChild(job.getId(), "AGGREGATE", Map.of("dataKey", dataKey));
        jobSubmissionService.submitChild(job.getId(), "FETCH_IMAGES", Map.of("userId", job.getUserId()));

        return JobResult.success(dataKey);
    }
}

// Fan-in: wait for all parent jobs to complete before starting D
@Component
public class JobDependencyResolver {

    @EventListener
    public void onJobCompleted(JobCompletedEvent event) {
        Job completedJob = event.getJob();

        // Find jobs waiting on this one
        List<Job> dependents = jobRepo.findByParentJobId(completedJob.getId());
        for (Job dependent : dependents) {
            boolean allParentsDone = jobRepo.allParentsCompleted(dependent.getId());
            if (allParentsDone) {
                // All dependencies satisfied — enqueue dependent job
                kafkaTemplate.send("jobs-standard", dependent.getId(),
                        new JobEvent(dependent.getId(), dependent.getType()));
            }
        }
    }
}
```

**For complex DAGs — use Temporal (covered in multi-step processes pattern):**

```java
// When job dependency graphs are complex, Temporal handles it elegantly
@WorkflowImpl
public class ReportGenerationWorkflow {

    @Override
    public void generate(String userId) {
        // Parallel fan-out
        Promise<String> dataFuture = Async.function(activities::fetchData, userId);
        Promise<String> imagesFuture = Async.function(activities::fetchImages, userId);

        // Fan-in: wait for both
        String dataKey = dataFuture.get();
        String imagesKey = imagesFuture.get();

        // Sequential step after both complete
        String reportUrl = activities.generateReport(dataKey, imagesKey);
        activities.notifyUser(userId, reportUrl);
    }
}
```

---

## Real-World Examples

### Video Transcoding (YouTube-like)

```text
1. User uploads video → presigned multipart URL (blob pattern)
2. S3 event → SQS → submission service
3. Job created: type=VIDEO_TRANSCODE, payload={sourceKey, videoId}
4. Worker picks up job:
   a. Download from S3 (progress: 10%)
   b. Transcode to 360p (progress: 30%)
   c. Transcode to 720p (progress: 55%)
   d. Transcode to 1080p (progress: 75%)
   e. Generate thumbnail (progress: 85%)
   f. Upload HLS segments to S3 (progress: 95%)
   g. Update video metadata: status=PUBLISHED (progress: 100%)
5. User receives SSE notification: "Your video is live!"

Failure handling:
  - Transcode fails → retry with same source, resume from last quality tier
  - Worker crashes → SQS visibility timeout expires → another worker picks up
  - All retries fail → DLQ → alert → human inspects source file
```

---

### Bulk Email Newsletter

```text
1. Admin clicks "Send newsletter to 2M users"
2. Web server: create job (PENDING), enqueue → Kafka
3. Worker:
   a. Fetch user IDs in batches of 10K (progress updates every batch)
   b. For each batch: call SES batch API (up to 50 emails per API call)
   c. Track per-user delivery status in Redis
   d. On completion: persist delivery stats to DB, update job COMPLETED

Mixed workload routing:
  - Newsletter → "jobs-batch" queue (5 workers, no rush)
  - Transactional email (password reset) → "jobs-critical" queue (50 workers, SLA 5s)

Backpressure:
  - SES rate limit: 14 emails/second by default
  - Worker sleeps between batches to respect rate limit
  - 2M emails / 14 per second = ~40 hours (expected — batches spread across time)
```

---

### PDF Report Generation

```text
1. POST /api/reports/generate → jobId returned immediately
2. Worker executes:
   a. Query DB for user's annual activity (millions of rows aggregated)
   b. Compute charts (CPU intensive)
   c. Render PDF (memory intensive)
   d. Upload to S3 → presigned download URL generated
   e. Job status = COMPLETED, resultUrl = presigned URL (1h expiry)
3. Client polling /api/jobs/{jobId} every 2s sees progress: 0 → 25 → 60 → 90 → 100
4. On 100%: client shows download button

Failure:
  - DB query times out → retry (data unchanged, query is idempotent)
  - PDF rendering OOM → worker restarted, job retried with fresh memory
  - 3 failures → DLQ → user notified "Report generation failed, try again"
```

---

## Interview Cheat Sheet

### Core Pattern Summary

```text
Long-running task:
  POST /api/tasks → validate → create job in DB → enqueue → return jobId

Client:
  GET /api/jobs/{jobId} every N seconds → poll until COMPLETED or FAILED

Worker:
  Dequeue → claim job (atomic UPDATE) → execute → update status → notify
```

---

### When to Use

- Task takes > 5 seconds (or > 1 second under high load)
- Task can fail and needs retry without client re-submission
- User needs progress feedback during execution
- Task needs to survive server restarts
- Multiple dependent steps that must be orchestrated

### When NOT to Use

- Task completes in < 1 second → just return result synchronously
- Task result is needed immediately in the same flow (use sync or streaming)
- Simple CRUD with no heavy computation

---

### Queue Choice Guide

| Requirement | Choose |
|-------------|--------|
| High throughput, replay needed | Kafka |
| Priority queues, flexible routing | RabbitMQ |
| AWS-native, simple | SQS |
| Ordered + exactly-once | SQS FIFO |
| Simple, low scale | Redis List |
| Complex DAG dependencies | Temporal |

---

### Common Deep Dive Answers

**"How do you handle job failures?"**

- Transient: retry with exponential backoff + jitter (3–5 attempts)
- Permanent: move to DLQ after max retries, alert oncall, notify user
- Partial: checkpoint completed steps, resume from last checkpoint on retry

**"How do you prevent duplicate job execution?"**

- Atomic `UPDATE ... WHERE status = PENDING` to claim a job (only one worker wins)
- Idempotency key on job submission (client-generated UUID)
- Make job handlers idempotent: upsert results, check-before-act

**"How do you handle queue backpressure?"**

- Auto-scale workers based on queue depth metric
- Rate limit job submission when queue exceeds threshold
- Separate queues per priority — batch jobs don't block critical ones
- Expire stale PENDING jobs after TTL

**"How do you handle mixed workloads?"**

- Separate queues: critical / standard / batch
- Different worker pool sizes per queue (50 / 20 / 5)
- Route at submission time based on job type
- SLA: critical < 5s, standard < 60s, batch < 1h

**"How do you orchestrate dependent jobs?"**

- Simple chain: on completion, enqueue next job
- Fan-out/fan-in: track parent job IDs, enqueue when all parents complete
- Complex DAG: use Temporal workflow engine

---

### Architecture Decision Flowchart

```text
Task takes > 5 seconds?
          │
          ▼
   YES → Async job pattern (this pattern)
   NO  → Synchronous response
          │
          ▼
Task has multiple dependent steps?
   YES → Checkpoint each step (resume on retry)
   NO  → Single atomic execution
          │
          ▼
Multiple job types with different priorities?
   YES → Separate queues per priority tier
   NO  → Single queue
          │
          ▼
Jobs form complex DAG dependencies?
   YES → Temporal workflow engine
   NO  → Chain jobs via completion events
          │
          ▼
High throughput (>10K jobs/sec) or need replay?
   YES → Kafka
   NO  → SQS / RabbitMQ / Redis
```

---

### Key Terms Quick Reference

| Term | Meaning |
|------|---------|
| **Job** | Unit of async work with status, payload, result |
| **Queue** | Buffer between job producers and workers |
| **Worker** | Process that consumes jobs from queue and executes them |
| **DLQ** | Dead Letter Queue — receives jobs that failed all retries |
| **Visibility timeout** | SQS: time a message is hidden from other consumers while being processed |
| **Idempotency key** | Client-generated key to prevent duplicate job submission |
| **Claim** | Atomic DB update to assign a job to one worker |
| **Checkpoint** | Persist intermediate step result so retry can resume |
| **Backpressure** | Queue growing faster than workers can consume |
| **Fan-out** | One job spawning multiple child jobs |
| **Fan-in** | Job waiting for multiple parent jobs to complete |
| **Priority queue** | Queue that dequeues higher-priority items first |

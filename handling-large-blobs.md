# Handling Large Blobs — System Design Pattern

## Table of Contents

1. [The Problem](#the-problem)
2. [Why Not Route Through App Servers](#why-not-route-through-app-servers)
3. [Solution Overview](#solution-overview)
4. [Simple Direct Upload](#simple-direct-upload)
5. [Simple Direct Download](#simple-direct-download)
6. [Resumable Uploads for Large Files](#resumable-uploads-for-large-files)
   - [Multipart Upload](#multipart-upload)
   - [State Synchronization](#state-synchronization)
7. [CDN for Fast Downloads](#cdn-for-fast-downloads)
8. [Deep Dives](#deep-dives)
   - [Upload Fails at 99%](#upload-fails-at-99)
   - [Preventing Abuse](#preventing-abuse)
   - [Handling Metadata](#handling-metadata)
   - [Ensuring Fast Downloads](#ensuring-fast-downloads)
9. [Cloud Provider Terminology](#cloud-provider-terminology)
10. [Real-World Examples](#real-world-examples)
11. [Interview Cheat Sheet](#interview-cheat-sheet)

---

## The Problem

Large files (videos, images, documents) need special handling. The naive approach routes file bytes through your application servers:

```text
NAIVE — App server as proxy:

Client ──upload 2GB video──▶ App Server ──forward to S3──▶ S3
Client ◀──download 2GB──── App Server ◀──stream from S3── S3

Problems:
  - App server CPU/memory consumed receiving and forwarding gigabytes
  - App server becomes bottleneck — 100 concurrent 2GB uploads = 200GB in RAM
  - Network bandwidth of app server exhausted
  - Timeouts on large files through HTTP proxies
  - App servers can't scale independently from file transfer capacity
```

**Rule of thumb:** If it's over 10MB and doesn't need SQL queries, it belongs in blob storage — and transfers should bypass app servers entirely.

---

## Why Not Route Through App Servers

```text
App server memory:     typically 4–16GB
Concurrent uploads:    100 users × 2GB video = 200GB needed in RAM → impossible

App server bandwidth:  typically 1–10 Gbps
100 concurrent 2GB uploads: 200GB × 8 bits = 1.6 Tbits → impossible

S3 bandwidth:          effectively unlimited (AWS backbone)
S3 throughput:         5,500 requests/sec per prefix, unlimited storage
```

The fix: **presigned URLs** — let clients talk directly to blob storage, bypassing app servers for the actual bytes.

---

## Solution Overview

```text
CORRECT — Direct client-to-storage transfer:

         1. Request upload URL
Client ──────────────────────▶ App Server
         2. Return presigned URL
Client ◀────────────────────── App Server
         3. Upload directly to S3
Client ──────────────────────────────────▶ S3
         4. Notify app server (upload complete)
Client ──────────────────────▶ App Server
         5. App server updates metadata in DB
                               App Server ──▶ DB

App server never touches the file bytes.
```

---

## Simple Direct Upload

### Step 1 — Client requests a presigned upload URL

```java
@RestController
@RequestMapping("/api/uploads")
public class UploadController {

    private final S3Presigner presigner;
    private final FileMetadataRepository metadataRepo;

    @PostMapping("/presigned-url")
    public PresignedUploadResponse getUploadUrl(@RequestBody UploadRequest req,
                                                 @AuthenticationPrincipal User user) {
        String objectKey = "uploads/" + user.getId() + "/" + UUID.randomUUID() + "/" + req.getFileName();

        // Generate presigned PUT URL — valid for 15 minutes
        PutObjectPresignRequest presignRequest = PutObjectPresignRequest.builder()
                .signatureDuration(Duration.ofMinutes(15))
                .putObjectRequest(PutObjectRequest.builder()
                        .bucket("my-app-uploads")
                        .key(objectKey)
                        .contentType(req.getContentType())
                        .contentLength(req.getFileSizeBytes())
                        .build())
                .build();

        PresignedPutObjectRequest presignedRequest = presigner.presignPutObject(presignRequest);

        // Store pending metadata in DB
        FileMetadata metadata = FileMetadata.builder()
                .id(UUID.randomUUID().toString())
                .objectKey(objectKey)
                .userId(user.getId())
                .fileName(req.getFileName())
                .fileSizeBytes(req.getFileSizeBytes())
                .status(FileStatus.PENDING)
                .build();
        metadataRepo.save(metadata);

        return PresignedUploadResponse.builder()
                .uploadUrl(presignedRequest.url().toString())
                .fileId(metadata.getId())
                .expiresAt(Instant.now().plus(Duration.ofMinutes(15)))
                .build();
    }
}
```

### Step 2 — Client uploads directly to S3

```text
Client sends:
  PUT https://my-app-uploads.s3.amazonaws.com/uploads/user123/uuid/video.mp4
      ?X-Amz-Algorithm=AWS4-HMAC-SHA256
      &X-Amz-Credential=...
      &X-Amz-Date=...
      &X-Amz-Expires=900
      &X-Amz-Signature=...

Body: [raw file bytes — goes directly to S3, never touches app server]

Response: HTTP 200 OK (from S3 directly)
```

### Step 3 — Client notifies app server of completion

```java
@PostMapping("/{fileId}/complete")
public ResponseEntity<FileMetadata> confirmUpload(@PathVariable String fileId,
                                                   @AuthenticationPrincipal User user) {
    FileMetadata metadata = metadataRepo.findById(fileId)
            .orElseThrow(() -> new NotFoundException(fileId));

    // Verify file actually exists in S3 (don't trust client blindly)
    HeadObjectResponse s3Object = s3Client.headObject(HeadObjectRequest.builder()
            .bucket("my-app-uploads")
            .key(metadata.getObjectKey())
            .build());

    if (s3Object.contentLength() != metadata.getFileSizeBytes()) {
        throw new BadRequestException("File size mismatch");
    }

    metadata.setStatus(FileStatus.READY);
    metadata.setUploadedAt(Instant.now());
    metadataRepo.save(metadata);

    // Trigger async processing (e.g., video transcoding)
    eventPublisher.publishEvent(new FileUploadedEvent(metadata));

    return ResponseEntity.ok(metadata);
}
```

**Alternatively — use S3 event notifications instead of client callback:**

```java
// S3 triggers this via SQS/SNS when upload completes — no client callback needed
@SqsListener("upload-completions-queue")
public void onS3UploadComplete(S3EventNotification event) {
    event.getRecords().forEach(record -> {
        String objectKey = record.getS3().getObject().getKey();
        long fileSize = record.getS3().getObject().getSizeAsLong();

        metadataRepo.findByObjectKey(objectKey).ifPresent(metadata -> {
            metadata.setStatus(FileStatus.READY);
            metadata.setUploadedAt(Instant.now());
            metadataRepo.save(metadata);
            eventPublisher.publishEvent(new FileUploadedEvent(metadata));
        });
    });
}
```

---

## Simple Direct Download

Client downloads directly from S3 or CDN using a presigned URL or CDN-signed URL.

### Presigned GET URL (for private files)

```java
@GetMapping("/{fileId}/download-url")
public DownloadUrlResponse getDownloadUrl(@PathVariable String fileId,
                                           @AuthenticationPrincipal User user) {
    FileMetadata metadata = metadataRepo.findById(fileId)
            .orElseThrow(() -> new NotFoundException(fileId));

    // Authorization check
    if (!metadata.getUserId().equals(user.getId()) && !user.hasRole("ADMIN")) {
        throw new ForbiddenException("Access denied");
    }

    // Generate presigned GET URL — valid for 1 hour
    GetObjectPresignRequest presignRequest = GetObjectPresignRequest.builder()
            .signatureDuration(Duration.ofHours(1))
            .getObjectRequest(GetObjectRequest.builder()
                    .bucket("my-app-uploads")
                    .key(metadata.getObjectKey())
                    .responseContentDisposition("attachment; filename=\"" + metadata.getFileName() + "\"")
                    .build())
            .build();

    PresignedGetObjectRequest presignedRequest = presigner.presignGetObject(presignRequest);

    return DownloadUrlResponse.builder()
            .downloadUrl(presignedRequest.url().toString())
            .expiresAt(Instant.now().plus(Duration.ofHours(1)))
            .fileName(metadata.getFileName())
            .fileSizeBytes(metadata.getFileSizeBytes())
            .build();
}
```

### CDN-signed URL (for public/semi-public files like YouTube videos)

```java
@GetMapping("/{fileId}/stream-url")
public StreamUrlResponse getStreamUrl(@PathVariable String fileId) {

    FileMetadata metadata = metadataRepo.findById(fileId)
            .orElseThrow(() -> new NotFoundException(fileId));

    // Generate CloudFront signed URL — routed through CDN edge nodes
    String cdnUrl = cloudFrontSigner.sign(
            "https://cdn.myapp.com/" + metadata.getObjectKey(),
            Instant.now().plus(Duration.ofHours(6))
    );

    return StreamUrlResponse.builder()
            .streamUrl(cdnUrl)
            .fileName(metadata.getFileName())
            .build();
}
```

---

## Resumable Uploads for Large Files

For files over ~100MB, a single PUT can fail partway through. Resumable uploads split the file into chunks and track which chunks have been received.

### Multipart Upload

S3 supports multipart upload natively — file is split into parts (min 5MB each), uploaded in parallel, then assembled by S3.

```text
Large file upload flow:

1. Initiate multipart upload → S3 returns uploadId
2. Upload part 1 (5MB) → S3 returns ETag for part 1
3. Upload part 2 (5MB) → S3 returns ETag for part 2
4. Upload part N (last part, any size) → S3 returns ETag
5. Complete multipart → S3 assembles all parts into final object

If part 3 fails:
  → Retry only part 3 (parts 1, 2 already safely on S3)
  → No need to re-upload entire file
```

**App server initiates multipart and returns presigned URLs for each part:**

```java
@PostMapping("/multipart/initiate")
public MultipartInitiateResponse initiateMultipartUpload(@RequestBody MultipartInitiateRequest req,
                                                          @AuthenticationPrincipal User user) {
    String objectKey = "uploads/" + user.getId() + "/" + UUID.randomUUID() + "/" + req.getFileName();

    // Initiate multipart upload
    CreateMultipartUploadResponse initResponse = s3Client.createMultipartUpload(
            CreateMultipartUploadRequest.builder()
                    .bucket("my-app-uploads")
                    .key(objectKey)
                    .contentType(req.getContentType())
                    .build());

    String uploadId = initResponse.uploadId();

    // Calculate part count (5MB parts minimum)
    long partSize = 5 * 1024 * 1024; // 5MB
    int partCount = (int) Math.ceil((double) req.getFileSizeBytes() / partSize);

    // Generate presigned URL for each part
    List<PartUploadUrl> partUrls = new ArrayList<>();
    for (int partNumber = 1; partNumber <= partCount; partNumber++) {
        UploadPartPresignRequest presignRequest = UploadPartPresignRequest.builder()
                .signatureDuration(Duration.ofHours(2))
                .uploadPartRequest(UploadPartRequest.builder()
                        .bucket("my-app-uploads")
                        .key(objectKey)
                        .uploadId(uploadId)
                        .partNumber(partNumber)
                        .build())
                .build();

        PresignedUploadPartRequest presigned = presigner.presignUploadPart(presignRequest);

        partUrls.add(PartUploadUrl.builder()
                .partNumber(partNumber)
                .uploadUrl(presigned.url().toString())
                .build());
    }

    // Persist upload session
    UploadSession session = UploadSession.builder()
            .sessionId(UUID.randomUUID().toString())
            .objectKey(objectKey)
            .uploadId(uploadId)
            .userId(user.getId())
            .totalParts(partCount)
            .completedParts(new HashSet<>())
            .status(UploadStatus.IN_PROGRESS)
            .expiresAt(Instant.now().plus(Duration.ofHours(24)))
            .build();
    sessionRepo.save(session);

    return MultipartInitiateResponse.builder()
            .sessionId(session.getSessionId())
            .uploadId(uploadId)
            .partUrls(partUrls)
            .partSizeBytes(partSize)
            .build();
}
```

**Client tracks part completion and notifies app server:**

```java
@PostMapping("/multipart/{sessionId}/part-complete")
public ResponseEntity<Void> markPartComplete(@PathVariable String sessionId,
                                              @RequestBody PartCompleteRequest req) {
    UploadSession session = sessionRepo.findById(sessionId)
            .orElseThrow(() -> new NotFoundException(sessionId));

    session.getCompletedParts().add(CompletedPartInfo.builder()
            .partNumber(req.getPartNumber())
            .etag(req.getEtag()) // ETag returned by S3 when part was uploaded
            .build());
    sessionRepo.save(session);

    return ResponseEntity.ok().build();
}

@PostMapping("/multipart/{sessionId}/complete")
public ResponseEntity<FileMetadata> completeMultipartUpload(@PathVariable String sessionId) {
    UploadSession session = sessionRepo.findById(sessionId)
            .orElseThrow(() -> new NotFoundException(sessionId));

    // Verify all parts received
    if (session.getCompletedParts().size() != session.getTotalParts()) {
        throw new BadRequestException("Not all parts uploaded yet");
    }

    // Tell S3 to assemble all parts
    List<CompletedPart> completedParts = session.getCompletedParts().stream()
            .sorted(Comparator.comparingInt(CompletedPartInfo::getPartNumber))
            .map(p -> CompletedPart.builder()
                    .partNumber(p.getPartNumber())
                    .eTag(p.getEtag())
                    .build())
            .collect(Collectors.toList());

    s3Client.completeMultipartUpload(CompleteMultipartUploadRequest.builder()
            .bucket("my-app-uploads")
            .key(session.getObjectKey())
            .uploadId(session.getUploadId())
            .multipartUpload(CompletedMultipartUpload.builder()
                    .parts(completedParts)
                    .build())
            .build());

    // Mark file as ready
    FileMetadata metadata = FileMetadata.builder()
            .objectKey(session.getObjectKey())
            .status(FileStatus.READY)
            .build();
    metadataRepo.save(metadata);
    sessionRepo.delete(session);

    return ResponseEntity.ok(metadata);
}
```

---

### State Synchronization Challenges

**Problem 1: Client loses track of which parts were uploaded (browser crash, network drop)**

**Solution:** App server stores completed parts. On resume, client queries which parts are done.

```java
@GetMapping("/multipart/{sessionId}/status")
public UploadStatusResponse getUploadStatus(@PathVariable String sessionId) {
    UploadSession session = sessionRepo.findById(sessionId)
            .orElseThrow(() -> new NotFoundException(sessionId));

    return UploadStatusResponse.builder()
            .sessionId(sessionId)
            .totalParts(session.getTotalParts())
            .completedPartNumbers(
                session.getCompletedParts().stream()
                       .map(CompletedPartInfo::getPartNumber)
                       .collect(Collectors.toSet()))
            .status(session.getStatus())
            .build();
    // Client re-uploads only the missing part numbers
}
```

**Problem 2: Incomplete multipart uploads accumulate in S3 (cost)**

```java
// S3 lifecycle rule — auto-abort incomplete multipart uploads after 24h
@PostConstruct
public void configureS3Lifecycle() {
    s3Client.putBucketLifecycleConfiguration(PutBucketLifecycleConfigurationRequest.builder()
            .bucket("my-app-uploads")
            .lifecycleConfiguration(BucketLifecycleConfiguration.builder()
                    .rules(LifecycleRule.builder()
                            .id("abort-incomplete-multipart")
                            .status(ExpirationStatus.ENABLED)
                            .abortIncompleteMultipartUpload(
                                AbortIncompleteMultipartUpload.builder()
                                    .daysAfterInitiation(1)
                                    .build())
                            .build())
                    .build())
            .build());
}
```

**Problem 3: Presigned URLs expire before upload completes (slow connection)**

```java
// Client detects expiry and requests fresh URLs for remaining parts
@PostMapping("/multipart/{sessionId}/refresh-urls")
public List<PartUploadUrl> refreshPartUrls(@PathVariable String sessionId,
                                            @RequestBody RefreshUrlsRequest req) {
    UploadSession session = sessionRepo.findById(sessionId)
            .orElseThrow(() -> new NotFoundException(sessionId));

    // Generate fresh presigned URLs only for remaining (not yet completed) parts
    Set<Integer> completedPartNumbers = session.getCompletedParts().stream()
            .map(CompletedPartInfo::getPartNumber)
            .collect(Collectors.toSet());

    return req.getPartNumbers().stream()
            .filter(p -> !completedPartNumbers.contains(p))
            .map(partNumber -> generatePresignedPartUrl(session, partNumber))
            .collect(Collectors.toList());
}
```

---

## CDN for Fast Downloads

Presigned S3 URLs serve from a single region. CDN serves from edge nodes globally.

```text
Without CDN:
  User in Tokyo ──download 2GB──▶ S3 (us-east-1) ──2GB cross-ocean──▶ User
  Latency: 150ms RTT × many round trips = slow

With CDN:
  User in Tokyo ──download 2GB──▶ CDN edge (Tokyo) ──cache hit──▶ User (fast)
                                                    ──miss──▶ S3 (us-east-1)──▶ edge ──▶ User (once)
  Latency: 5ms RTT from edge
```

**When to use CDN vs presigned URL:**

| Scenario | Use |
|----------|-----|
| Private file, one-time download | Presigned S3 URL |
| Public content, many users | CDN-signed URL |
| Video streaming | CDN (HLS/DASH segments cached at edge) |
| Profile images, thumbnails | CDN (public, long TTL) |
| Documents (contracts, invoices) | Presigned S3 URL (private, short TTL) |

**Video streaming with CDN (HLS):**

```text
Upload flow:
  Raw video → S3 → transcoding job (FFmpeg) → HLS segments (.ts files + .m3u8 playlist)
           → push segments to CDN origin → CDN distributes to edge nodes

Download flow:
  Client requests playlist → CDN edge returns .m3u8 (from cache)
  Client requests segment 1 → CDN edge returns .ts (from cache, <5ms)
  Client requests segment 2 → CDN edge returns .ts (from cache)
  ...

Result: video plays from local edge, no S3 round trip per segment
```

---

## Deep Dives

### Upload Fails at 99%

**Problem:** 2GB video upload fails at 99% — client has to start over.

**Solution: Multipart upload with checkpointing**

```text
File split into 400 × 5MB parts.
Parts 1–396 uploaded successfully (checkpointed in DB).
Part 397 fails.

On retry:
  Client queries /multipart/{sessionId}/status
  Server returns: completedParts = [1..396]
  Client re-uploads only part 397, 398, 399, 400
  → Re-uploads 20MB instead of 2GB (99% saved)
```

**Client-side retry logic (conceptual):**

```text
1. GET /multipart/{sessionId}/status → completed parts: [1..396]
2. Determine missing: [397, 398, 399, 400]
3. POST /multipart/{sessionId}/refresh-urls for [397..400] (old URLs expired)
4. Parallel upload of parts 397..400
5. POST /multipart/{sessionId}/part-complete for each
6. POST /multipart/{sessionId}/complete
```

**Exponential backoff for part retries:**

```java
@Service
public class PartUploadRetryService {

    public void uploadPartWithRetry(String presignedUrl, byte[] partData, int partNumber) {
        int maxAttempts = 5;
        long delayMs = 1000;

        for (int attempt = 1; attempt <= maxAttempts; attempt++) {
            try {
                httpClient.put(presignedUrl, partData); // PUT directly to S3
                return;
            } catch (Exception e) {
                if (attempt == maxAttempts) throw new UploadException("Part " + partNumber + " failed", e);
                Thread.sleep(delayMs * (1L << (attempt - 1))); // 1s, 2s, 4s, 8s, 16s
            }
        }
    }
}
```

---

### Preventing Abuse

Without controls, a malicious user could:

- Upload 1TB files (cost abuse)
- Upload malicious executables
- Spam uploads to exhaust storage

**Controls:**

```java
@PostMapping("/presigned-url")
public PresignedUploadResponse getUploadUrl(@RequestBody UploadRequest req,
                                             @AuthenticationPrincipal User user) {
    // 1. File size limit
    long maxSizeBytes = 5L * 1024 * 1024 * 1024; // 5GB
    if (req.getFileSizeBytes() > maxSizeBytes) {
        throw new BadRequestException("File exceeds maximum size of 5GB");
    }

    // 2. MIME type allowlist
    Set<String> allowedTypes = Set.of("video/mp4", "video/quicktime", "image/jpeg",
                                       "image/png", "application/pdf");
    if (!allowedTypes.contains(req.getContentType())) {
        throw new BadRequestException("File type not allowed: " + req.getContentType());
    }

    // 3. Rate limit per user
    if (!rateLimiter.allowUpload(user.getId())) {
        throw new TooManyRequestsException("Upload rate limit exceeded");
    }

    // 4. User quota check
    long usedStorage = metadataRepo.sumFileSizeByUserId(user.getId());
    long quotaBytes = user.getStorageQuotaBytes(); // e.g., 15GB for free tier
    if (usedStorage + req.getFileSizeBytes() > quotaBytes) {
        throw new BadRequestException("Storage quota exceeded");
    }

    // 5. Presigned URL with content-length condition — S3 rejects if size doesn't match
    PutObjectRequest putRequest = PutObjectRequest.builder()
            .bucket("my-app-uploads")
            .key(objectKey)
            .contentType(req.getContentType())
            .contentLength(req.getFileSizeBytes()) // S3 enforces this
            .build();

    // ... generate presigned URL
}
```

**Post-upload virus scanning:**

```java
// Triggered by S3 event after upload completes
@SqsListener("upload-completions-queue")
public void scanUpload(S3EventNotification event) {
    event.getRecords().forEach(record -> {
        String objectKey = record.getS3().getObject().getKey();

        // Quarantine immediately — move to unscanned prefix
        s3Client.copyObject(CopyObjectRequest.builder()
                .sourceBucket("my-app-uploads")
                .sourceKey(objectKey)
                .destinationBucket("my-app-quarantine")
                .destinationKey(objectKey)
                .build());
        s3Client.deleteObject(DeleteObjectRequest.builder()
                .bucket("my-app-uploads")
                .key(objectKey)
                .build());

        // Async scan (ClamAV, AWS GuardDuty Malware Protection)
        ScanResult result = virusScanner.scan("my-app-quarantine", objectKey);

        if (result.isClean()) {
            // Move to approved bucket
            s3Client.copyObject(... .destinationBucket("my-app-approved") ...);
            metadataRepo.updateStatus(objectKey, FileStatus.READY);
        } else {
            metadataRepo.updateStatus(objectKey, FileStatus.REJECTED);
            alertService.notifyMaliciousUpload(objectKey);
        }
    });
}
```

---

### Handling Metadata

Metadata (file name, size, owner, tags, status) lives in your DB — blob storage only holds the bytes.

```java
@Entity
@Table(name = "file_metadata")
public class FileMetadata {

    @Id
    private String id;

    private String objectKey;        // S3 key: "uploads/user123/uuid/video.mp4"
    private String bucket;           // "my-app-approved"
    private String fileName;         // original filename from user
    private String contentType;      // "video/mp4"
    private Long fileSizeBytes;

    @Enumerated(EnumType.STRING)
    private FileStatus status;       // PENDING, SCANNING, READY, REJECTED

    private String userId;
    private String folderId;         // logical grouping

    // Derived/processed file references
    private String thumbnailKey;     // S3 key for generated thumbnail
    private String transcodedKey;    // S3 key for transcoded video

    private Instant uploadedAt;
    private Instant expiresAt;       // for temporary files

    @ElementCollection
    private Map<String, String> customMetadata; // user-defined tags
}
```

**Soft delete — never delete from S3 immediately:**

```java
@DeleteMapping("/{fileId}")
public ResponseEntity<Void> deleteFile(@PathVariable String fileId,
                                        @AuthenticationPrincipal User user) {
    FileMetadata metadata = metadataRepo.findById(fileId)
            .orElseThrow(() -> new NotFoundException(fileId));

    // Soft delete — mark in DB, keep in S3 for recovery window
    metadata.setStatus(FileStatus.DELETED);
    metadata.setDeletedAt(Instant.now());
    metadataRepo.save(metadata);

    // Schedule hard delete from S3 after 30 days
    s3LifecycleService.scheduleDelete(metadata.getObjectKey(), Duration.ofDays(30));

    return ResponseEntity.noContent().build();
}
```

**Search across metadata:**

```java
// Metadata in DB enables rich queries — impossible with blob storage alone
@Query("SELECT f FROM FileMetadata f WHERE f.userId = :userId " +
       "AND f.status = 'READY' " +
       "AND (:contentType IS NULL OR f.contentType = :contentType) " +
       "AND f.uploadedAt >= :since " +
       "ORDER BY f.uploadedAt DESC")
Page<FileMetadata> searchFiles(@Param("userId") String userId,
                                @Param("contentType") String contentType,
                                @Param("since") Instant since,
                                Pageable pageable);
```

---

### Ensuring Fast Downloads

**Multi-region replication:**

```java
// Replicate to multiple S3 regions — serve from closest region
@PostConstruct
public void configureReplication() {
    s3Client.putBucketReplication(PutBucketReplicationRequest.builder()
            .bucket("my-app-approved-us-east-1")
            .replicationConfiguration(ReplicationConfiguration.builder()
                    .rules(ReplicationRule.builder()
                            .status(ReplicationRuleStatus.ENABLED)
                            .destination(Destination.builder()
                                    .bucket("arn:aws:s3:::my-app-approved-ap-southeast-1")
                                    .build())
                            .build())
                    .build())
            .build());
}
```

**Parallel download (for very large files):**

```text
File: 10GB

Client splits into 10 × 1GB byte-range requests — download in parallel:
  GET /file with Range: bytes=0-1073741823         → part 1
  GET /file with Range: bytes=1073741824-2147483647 → part 2
  ...
  GET /file with Range: bytes=9663676416-10737418239 → part 10

All 10 requests fire simultaneously → 10x faster than sequential
```

```java
// App server: return metadata including byte ranges for parallel download
@GetMapping("/{fileId}/parallel-download")
public ParallelDownloadResponse getParallelDownloadInfo(@PathVariable String fileId) {
    FileMetadata metadata = metadataRepo.findById(fileId).orElseThrow(...);

    long partSize = 100 * 1024 * 1024; // 100MB parts
    int partCount = (int) Math.ceil((double) metadata.getFileSizeBytes() / partSize);

    List<ByteRangePart> parts = new ArrayList<>();
    for (int i = 0; i < partCount; i++) {
        long start = (long) i * partSize;
        long end = Math.min(start + partSize - 1, metadata.getFileSizeBytes() - 1);

        // Generate presigned URL per byte range
        GetObjectPresignRequest presignRequest = GetObjectPresignRequest.builder()
                .signatureDuration(Duration.ofHours(1))
                .getObjectRequest(GetObjectRequest.builder()
                        .bucket(metadata.getBucket())
                        .key(metadata.getObjectKey())
                        .range("bytes=" + start + "-" + end)
                        .build())
                .build();

        parts.add(ByteRangePart.builder()
                .partNumber(i + 1)
                .downloadUrl(presigner.presignGetObject(presignRequest).url().toString())
                .startByte(start)
                .endByte(end)
                .build());
    }

    return ParallelDownloadResponse.builder()
            .fileId(fileId)
            .totalSizeBytes(metadata.getFileSizeBytes())
            .parts(parts)
            .build();
}
```

---

## Cloud Provider Terminology

| Concept | AWS | GCP | Azure |
|---------|-----|-----|-------|
| Blob storage | S3 | Cloud Storage | Blob Storage |
| Presigned URL | Presigned URL | Signed URL | SAS URL |
| CDN | CloudFront | Cloud CDN | Azure CDN |
| Multipart upload | Multipart Upload | Resumable Upload | Block Blob upload |
| Event notification | S3 Event → SQS/SNS | Pub/Sub notification | Event Grid |
| Object lifecycle | S3 Lifecycle Policy | Object Lifecycle | Lifecycle Management |
| Access control | IAM + Bucket Policy | IAM + ACL | RBAC + SAS |

---

## Real-World Examples

### YouTube — Video Upload

```text
Upload pipeline:
  1. Client → app server: "I want to upload 2GB video"
  2. App server → S3: initiate multipart upload, return uploadId + part URLs
  3. Client → S3: upload 400 × 5MB parts in parallel (8 parallel connections)
  4. S3 → SQS: "upload complete" event
  5. Consumer: verify file, update metadata status = PROCESSING
  6. Transcoding farm: FFmpeg → 360p, 720p, 1080p, 4K HLS segments → S3
  7. CDN pre-warms: pushes HLS segments to edge nodes in all regions
  8. Metadata updated: status = READY, HLS manifest URL stored

Download (streaming):
  Client → CDN edge: GET .m3u8 playlist (from cache, <5ms)
  Client → CDN edge: GET .ts segments (from cache, adaptive bitrate)
  CDN handles 99% of bandwidth — S3 only serves cache misses
```

---

### Google Drive — Document Storage

```text
Upload:
  Small files (<5MB): single presigned PUT URL
  Large files (>5MB): resumable multipart upload with session token

Download:
  Private documents: presigned GET URL (short TTL, 1 hour)
  Shared documents: CDN-signed URL (longer TTL, cached at edge)

Metadata (DB):
  file_id, owner_id, name, mime_type, size, version, parent_folder_id, shared_with[]

Versioning:
  Each edit → new S3 object key (content-addressed: hash of content)
  Old versions retained for 30 days, metadata keeps version history
  S3 versioning enabled for accidental deletion recovery

Collaboration:
  File bytes in S3/CDN — immutable until next save
  Real-time edits (Google Docs): operational transform over WebSocket
  Periodic snapshot → new S3 version
```

---

### Dropbox — File Sync

```text
Content-addressed storage:
  File hash (SHA-256) = S3 key
  Same file uploaded by two users → stored once (deduplication)
  "Upload" may return instantly if hash already exists in S3

Delta sync (bandwidth optimization):
  On file change: compute block-level diff
  Upload only changed blocks, not entire file
  Client library tracks block hashes

Chunked upload for large files:
  4MB chunks, parallel upload
  Server tracks which chunks are committed per file version
  Resume: re-upload only uncommitted chunks
```

---

## Interview Cheat Sheet

### Core Pattern Summary

```text
Always use presigned URLs — app servers should never proxy file bytes

Upload:
  Small (<100MB)  → single presigned PUT URL
  Large (>100MB)  → multipart upload (presigned URL per part)

Download:
  Private file    → presigned GET URL (short TTL)
  Public content  → CDN-signed URL (long TTL, edge cached)
  Video streaming → CDN + HLS (adaptive bitrate segments)
```

---

### When to Use Which Approach

| Scenario | Approach |
|----------|---------|
| Profile image upload | Single presigned PUT URL |
| Video upload (>100MB) | Multipart upload with part tracking |
| Private document download | Presigned GET URL (1h TTL) |
| Public video streaming | CDN + HLS |
| Thumbnail / public image | CDN (long TTL, public) |
| Resumable upload (mobile) | Multipart + server-side state |
| File deduplication | Content-addressed storage (SHA-256 key) |

---

### Common Deep Dive Answers

**"What if the upload fails at 99%?"**

- Multipart upload: only re-upload failed parts (tracked in DB)
- Client queries server for completed part list on resume
- Exponential backoff for part retries
- S3 lifecycle rule aborts orphaned multipart uploads after 24h

**"How do you prevent abuse?"**

- File size limit enforced at presigned URL generation and S3 content-length condition
- MIME type allowlist in presigned URL generation
- Per-user rate limiting on URL generation
- Storage quota check before issuing presigned URL
- Post-upload virus scan before marking file READY

**"How do you handle metadata?"**

- Blob storage holds bytes only — metadata always in relational DB
- DB enables rich queries, search, ACL checks, version history
- S3 event notification updates metadata status after upload
- Soft delete in DB, scheduled hard delete in S3 after 30 days

**"How do you ensure downloads are fast?"**

- CDN: serve from edge node closest to user (5ms vs 150ms cross-region)
- Multi-region S3 replication: serve from nearest region
- Parallel byte-range downloads: split large files into concurrent requests
- Video: HLS adaptive bitrate + CDN pre-warm for popular content
- Aggressive CDN TTL for public content (days/weeks)

---

### Architecture Decision Flowchart

```text
Need to handle file upload/download?
          │
          ▼
File > 10MB or binary content?
   YES → Use blob storage (S3), never DB
   NO  → Can store in DB as BYTEA if truly small
          │
          ▼
File > 100MB?
   YES → Multipart upload with server-side part tracking
   NO  → Single presigned PUT URL
          │
          ▼
Is content public or semi-public?
   YES → CDN-signed URL + long TTL
   NO  → Presigned GET URL + short TTL (1h)
          │
          ▼
Is it video?
   YES → Transcode to HLS, serve segments via CDN
   NO  → Direct download via presigned/CDN URL
          │
          ▼
Need upload resumability (mobile, unreliable network)?
   YES → Multipart + server tracks completed parts
   NO  → Single upload with client-side retry
```

---

### Key Terms Quick Reference

| Term | Meaning |
|------|---------|
| **Presigned URL** | Time-limited URL that grants temporary access to a specific S3 object |
| **Multipart upload** | Split large file into parts, upload independently, S3 assembles |
| **CDN** | Content Delivery Network — caches content at global edge nodes |
| **HLS** | HTTP Live Streaming — video split into small .ts segments + .m3u8 playlist |
| **ETag** | S3's identifier for a specific version of an object (used in multipart completion) |
| **Byte-range request** | HTTP Range header — download a specific portion of a file |
| **Content-addressed** | Storage key = hash of content — enables deduplication |
| **SAS URL** | Azure equivalent of presigned URL |
| **Signed URL** | GCP equivalent of presigned URL |
| **Object lifecycle** | S3 rules to auto-delete or transition objects after N days |
| **S3 event notification** | S3 triggers SQS/SNS/Lambda when objects are created/deleted |

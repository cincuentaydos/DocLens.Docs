# ADR-006 — Synchronous vs Asynchronous Extraction Processing

**Status:** Superseded — decision changed to asynchronous (event-driven)

---

## Decision

**Process documents asynchronously via an SQS-driven worker Lambda.** `POST /documents/process` returns `202 Accepted` immediately. A separate processing Lambda consumes from SQS and performs Textract + Bedrock extraction. The client polls `GET /documents/{documentId}/versions/{versionNumber}` until status is no longer `PENDING`.

---

## Context

DocLens receives document processing requests via `POST /documents/process`. Once the file is in S3 and has passed the malware scan, two external service calls must complete before structured data can be returned: OCR (Textract) and semantic analysis (Bedrock). There are two architecturally distinct ways to handle this work: synchronously within the same Lambda invocation that received the request, or asynchronously via a job queue that decouples intake from processing.

The initial design chose synchronous processing for simplicity. This decision was revisited because the **API Gateway HTTP API v2 hard timeout of 29 seconds** cannot be configured or extended — it is an AWS-imposed limit. Textract on multi-page scanned documents, combined with Bedrock inference, regularly approaches or exceeds this ceiling. A `504 Gateway Timeout` at this layer means the extraction result is silently lost, with no retry possible at the infrastructure level.

---

## Options Considered

### Option 1 — Synchronous (inline) *(initial approach — rejected)*

Textract and Bedrock are invoked sequentially within the same Lambda execution that handled `POST /documents/process`. The HTTP response is held open until both calls return.

```
POST /documents/process
  → Lambda invoked
  → Textract (OCR)      ~1–5s typical, longer for scanned multi-page
  → Bedrock (Claude)    ~2–8s typical
  → HTTP 200 returned
```

**Strengths**

- Simple client integration — one request, one response, no polling.
- Simple backend — no queue, no job record, no separate worker Lambda.
- Straightforward error handling — failures surface immediately as HTTP errors.
- Easy to observe and debug — a single trace covers the entire operation.

**Weaknesses**

- **API Gateway timeout:** HTTP API v2 has a hard 29-second integration timeout. If Textract + Bedrock together exceed this the client gets a `504 Gateway Timeout` and the result is lost. Multi-page scanned documents routinely breach this limit.
- **No retry isolation:** if the Lambda crashes mid-extraction, the client gets a 5xx with no partial result and must retry the full operation.
- **Throughput ceiling:** a single Lambda instance handles one document at a time. High concurrency requires Lambda scaling, which adds cold starts.
- **GuardDuty polling pushed to client:** the client must implement exponential backoff on `409 Conflict` while the scan completes — leaking infrastructure detail into the frontend.

---

### Option 2 — Asynchronous (event-driven, SQS) *(chosen)*

`POST /documents/process` enqueues a job and returns `202 Accepted` immediately. A separate worker Lambda consumes the SQS queue and performs GuardDuty gate + SHA check + Textract + Bedrock. The client polls the existing version status endpoint.

```
POST /documents/process
  → intake Lambda enqueues message to SQS
  → HTTP 202 Accepted

S3 + GuardDuty scan (handled internally by worker)
SQS → Worker Lambda
  → GuardDuty gate (retry internally if tag absent)
  → SHA-256 check
  → Textract + Bedrock
  → writes COMPLETED / DUPLICATE / REJECTED to DynamoDB

GET /documents/{documentId}/versions/{versionNumber}
  → { status: "PENDING" | "COMPLETED" | "DUPLICATE" | "REJECTED", fields, diffFromPrevious }
```

**Strengths**

- **No API Gateway timeout constraint** — the 202 is returned before any extraction begins.
- **GuardDuty handled internally** — the worker Lambda retries the tag check internally; the client never sees a `409`.
- **Retry isolation** — SQS DLQ catches failures without client impact; failed documents are not silently lost.
- **Natural backpressure** — SQS buffers traffic spikes without dropping requests.
- **Enables Textract async jobs** (`StartDocumentTextDetection`) for large multi-page scanned documents without any architectural change.
- **Worker Lambda independently sized** — more memory and timeout headroom for heavy workloads without affecting the intake Lambda.
- **Simpler client** — frontend uploads, triggers processing, then polls one status endpoint. No knowledge of GuardDuty, Textract, or Bedrock required.

**Weaknesses**

- Slightly more complex infrastructure: SQS queue, DLQ, worker Lambda, IAM wiring.
- Harder to debug — a failed extraction must be traced across two Lambda invocations.
- Adds latency for typical fast documents (queue polling overhead, typically < 1s).

---

### Option 3 — Fully Event-Driven: S3 → SQS directly *(proposed — under discussion, not yet decided)*

Raised while reviewing ADR-005: instead of the client calling `POST /documents/process` to enqueue the job, an **S3 Event Notification** (`s3:ObjectCreated:Put`, filtered by suffix `.pdf`) publishes directly to the SQS queue the moment the pre-signed PUT completes. The worker Lambda is unchanged — it reads `documentType` from the `PENDING` DynamoDB record (looked up via `tenantId`/`documentId` parsed from the S3 key) instead of receiving it in a request body. The intake Lambda's `/process` endpoint is removed entirely; the client never calls it.

```
1. POST /documents/prepare      → { documentId, versionNumber, uploadUrl }
2. PUT {uploadUrl}               → 200 OK (direct S3)
3. S3 ObjectCreated:Put event    → SQS (no client call, no intake Lambda)
4. Worker Lambda consumes message
   → GuardDuty gate (tag usually absent on first attempt — SQS retries as today)
   → SHA-256 check → Textract + Bedrock → writes COMPLETED / DUPLICATE / REJECTED
5. Client polls GET /documents/{id}/versions/{n}
```

**Strengths**

- One fewer client round trip — upload alone is sufficient to trigger processing.
- Removes the intake Lambda / `/process` endpoint entirely — less code to own.
- Eliminates the "abandoned `PENDING` record" open question from [ADR-005](005-upload-strategy.md#open-questions) — there is no window where an upload completes but processing is never triggered.
- `tenantId` is already embedded in the S3 key (set server-side at `prepare` time) and `documentType` is already in the `PENDING` record — no data is lost by removing the request body.
- Duplicate or retried S3 events are already safe — the SHA-256 check ([ADR-007](007-document-versioning.md)) treats re-processing the same content as `DUPLICATE`.

**Weaknesses**

- GuardDuty tag-absent-on-first-attempt becomes the **common** case instead of the exception — the scan has barely started when the S3 event fires. Today, the client's extra round trip to call `/process` incidentally gives the scan a head start.
- The S3 Event Notification must be suffix-filtered to `*.pdf` — the companion `{documentId}.pdf.metadata.json` file (written for RAG ingestion) must not also trigger the worker.
- Loses the explicit client-side "commit" signal — no way for the client to delay or batch-trigger processing after uploading multiple files. Not a known requirement today, but worth confirming with the team.
- No request/response tied to the act of starting processing — client already has `documentId` from `prepare()`, so this is mostly cosmetic, but removes a natural place to fail fast on a malformed request.

---

## Comparison

| Dimension | Synchronous | Asynchronous (chosen) | Event-driven, no `/process` (proposed) |
|---|---|---|---|
| Client complexity | Low (one request) | Low (202 + status polling) | Lowest (upload + polling only) |
| API Gateway timeout | Hard 29s limit — unworkable for scanned PDFs | Not applicable | Not applicable |
| GuardDuty polling | Client-side (409 + backoff) | Internal to worker Lambda | Internal to worker Lambda (tag-absent retry is now the common path, not the exception) |
| Large scanned PDFs | Risk of 504 | Fully supported | Fully supported |
| Retry isolation | None | SQS DLQ | SQS DLQ |
| Infrastructure | Minimal | Queue + DLQ + worker Lambda | Queue + DLQ + worker Lambda, **no intake Lambda** |
| Observability | Single trace | Multi-invocation (X-Ray correlation) | Multi-invocation; no client-initiated span to correlate against |
| Robustness | Low — lost results on timeout | High — no silent failures | High — no silent failures, and no abandoned `PENDING` records |

---

## Chosen Approach

**Asynchronous, SQS-driven.** The API Gateway hard limit is the deciding factor — synchronous extraction cannot be made reliable for all document types without circumventing a limit that cannot be configured away.

### Client flow

```
1. POST /documents/prepare          → { documentId, versionNumber, uploadUrl }
2. PUT {uploadUrl}                  → 200 OK (direct S3)
3. POST /documents/process          → 202 Accepted
4. poll GET /documents/{id}/versions/{n}
   until status ≠ PENDING
   → COMPLETED   : fields and diffFromPrevious available
   → DUPLICATE   : content unchanged, refer to previous version
   → REJECTED    : file flagged by GuardDuty
```

### Infrastructure components

| Component | Role |
|---|---|
| Intake Lambda (`POST /documents/process`) | Validates request, enqueues SQS message, returns 202 |
| SQS queue | Buffers jobs; decouples intake from processing |
| SQS DLQ | Captures jobs that fail after max retries |
| Worker Lambda | GuardDuty gate, SHA check, OCR, Bedrock, DynamoDB write |
| `GET /documents/{id}/versions/{n}` | Status polling endpoint; reads DynamoDB directly |

### Worker Lambda — internal GuardDuty handling

The worker Lambda reads the GuardDuty tag on the S3 object. If the tag is absent (scan still running), the Lambda does **not** process the document — it raises an exception so SQS retries the message after the visibility timeout. This converts the client-facing `409` polling loop into an internal infrastructure concern.

### Retry and failure policy

| Scenario | Handling |
|---|---|
| GuardDuty tag absent at first attempt | SQS retry (visibility timeout ~30s) |
| THREATS_FOUND or UNSCANNABLE | Write REJECTED to DynamoDB; no retry |
| Textract / Bedrock transient failure | SQS retry (up to 3 attempts) |
| Worker Lambda crash | SQS retry; after max retries → DLQ |
| DLQ message | Operator alert; manual inspection |

---

## Migration Seam

The `IDocumentExtractionService` interface isolates the extraction logic from the endpoint layer. The intake Lambda calls `IDocumentExtractionService.EnqueueAsync(...)` (replacing the former `ExtractAsync(...)`). The worker Lambda calls the same underlying OCR and semantic services. No changes are required to `IOcrService` or `ISemanticAnalysisService`.

---

## Consequences

- **API Reference:** `POST /documents/process` now returns `202 Accepted` instead of `200 OK` with fields.
- **Lambda timeout:** the intake Lambda timeout can be reduced to ~10 seconds (queue write only). The worker Lambda is configured at **5 minutes** to accommodate long Textract jobs.
- **IAM:** intake Lambda needs `sqs:SendMessage`; worker Lambda needs `sqs:ReceiveMessage`, `sqs:DeleteMessage`, `sqs:GetQueueAttributes`.
- **Observability:** AWS X-Ray trace IDs must be propagated through the SQS message attributes to correlate intake and worker invocations in a single trace.
- **Frontend:** polling interval recommendation — start at 2s, back off to 5s after 10s, cap at 10s. Surface an error to the user after 5 minutes without a terminal status.

---

## Open Questions

- **Pending team discussion:** should `POST /documents/process` be removed in favor of the fully event-driven trigger described in [Option 3](#option-3-fully-event-driven-s3-sqs-directly-proposed-under-discussion-not-yet-decided) (S3 Event Notification → SQS directly)? This would remove the intake Lambda's `/process` endpoint and resolve the "abandoned `PENDING` record" open question in ADR-005, at the cost of making the GuardDuty tag-absent retry path the common case instead of the exception.

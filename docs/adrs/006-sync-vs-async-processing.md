# ADR-006 — Synchronous vs Asynchronous Extraction Processing

**Status:** Accepted — synchronous, with documented triggers for revisiting

---

## Decision

**Process documents synchronously within the Lambda invocation.** Textract and Bedrock are called inline; the HTTP response is returned only after extraction completes.

---

## Context

DocLens receives document processing requests via `POST /documents/process`. Once the file is in S3 and has passed the malware scan, two external service calls must complete before structured data can be returned: OCR (Textract) and semantic analysis (Bedrock). There are two architecturally distinct ways to handle this work: synchronously within the same Lambda invocation that received the request, or asynchronously via a job queue that decouples intake from processing.

This decision affects Lambda timeout configuration, API Gateway timeout limits, client integration complexity, cost, and operational observability.

---

## Options Considered

### Option 1 — Synchronous (inline)

Textract and Bedrock are invoked sequentially within the same Lambda execution that handled `POST /documents/process`. The HTTP response is held open until both calls return.

```
POST /documents/process
  → Lambda invoked
  → Textract (OCR)      ~1–5s typical
  → Bedrock (Claude)    ~2–8s typical
  → HTTP 200 returned
```

**Strengths**

- Simple client integration — one request, one response, no polling.
- Simple backend — no queue, no job record, no separate worker Lambda.
- Straightforward error handling — failures surface immediately as HTTP errors.
- Easy to observe and debug — a single trace covers the entire operation.
- Sufficient for document sizes and volumes typical at DocLens launch.

**Weaknesses**

- **API Gateway timeout:** HTTP API v2 has a hard 30-second integration timeout. If Textract + Bedrock together exceed 29 seconds the response is lost. Multi-page scanned documents via Textract async jobs are not compatible with this limit.
- **Lambda timeout:** must be configured generously (30–60 seconds) even for typical fast-path documents.
- **No retry isolation:** if the Lambda crashes mid-extraction (OOM, cold start timeout), the client gets a 5xx with no partial result and must retry the full operation.
- **Throughput ceiling:** a single Lambda instance handles one document at a time. High concurrency requires Lambda scaling, which adds cold starts.

---

### Option 2 — Asynchronous (queue-based)

`POST /documents/process` enqueues a job and returns immediately with `202 Accepted` and a `jobId`. A separate worker Lambda consumes the queue and performs extraction. The client polls a status endpoint or receives a webhook/SNS notification when done.

```
POST /documents/process
  → Lambda enqueues job to SQS
  → HTTP 202 Accepted { jobId }

SQS → Worker Lambda
  → Textract + Bedrock (no timeout pressure)
  → writes result to DynamoDB

GET /documents/{documentId}/status
  → returns { status: "processing" | "completed" | "failed", fields: ... }
```

**Strengths**

- No API Gateway or Lambda timeout constraint on extraction duration.
- Retry isolation — SQS dead-letter queue catches failures without client impact.
- Natural backpressure — SQS buffers traffic spikes without dropping requests.
- Enables Textract async jobs (`StartDocumentTextDetection`) for large multi-page scanned documents.
- Worker Lambda can be sized independently (more memory for heavy Textract workloads).

**Weaknesses**

- Significantly more complex client integration — requires polling or webhook handling.
- More infrastructure: SQS queue, dead-letter queue, worker Lambda, status endpoint.
- Harder to debug — a failed extraction must be traced across two Lambda invocations.
- Adds latency for typical fast documents (queue polling overhead).
- More moving parts to monitor and operate.

---

## Comparison

| Dimension | Synchronous | Asynchronous |
|---|---|---|
| Client complexity | Low (one request) | High (polling or webhook) |
| Infrastructure | Minimal | Queue + worker + status endpoint |
| API Gateway timeout | Hard 30s limit | Not applicable (202 returned immediately) |
| Large scanned PDFs | Risk of timeout | Fully supported |
| Retry isolation | None | SQS DLQ |
| Observability | Single trace | Multi-invocation tracing required |
| Time to implement | Low | High |
| Suitable for current scale | Yes | Premature |

---

## Chosen Approach

Synchronous processing is adopted for the current phase. The typical extraction path — PdfPig fast path or single-page Textract + Bedrock — completes well within the 30-second API Gateway limit. The added complexity of an async pipeline is not justified by the current document volume or size profile.

**Lambda timeout** is configured at **60 seconds** to accommodate Textract's synchronous call latency on moderate-length documents while staying above the API Gateway ceiling.

**Textract async jobs** (`StartDocumentTextDetection`) are not used in the synchronous path. Multi-page scanned documents that would require them are deferred to the async migration (see triggers below).

---

## Triggers for Revisiting

This decision should be revisited when **any** of the following conditions arise:

| Trigger | Threshold |
|---|---|
| Extraction p95 latency approaches API Gateway limit | > 25 seconds |
| Multi-page scanned documents become a primary use case | > 20% of volume |
| Client SLA requires sub-second acknowledgement | Product decision |
| Document volume requires independent scaling of intake vs. processing | > 50 concurrent extractions sustained |
| Textract async jobs are needed for accuracy on long documents | Determined by OCR quality review |

When migrating to async, the `IDocumentExtractionService` interface is the seam: the synchronous implementation can be replaced with a queue-dispatching implementation without changing the endpoint layer.

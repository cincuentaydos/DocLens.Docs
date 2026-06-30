# Data Flow

## Full Upload and Processing Flow

The complete flow from a client uploading a document to receiving extraction results spans three steps: prepare, upload, and process.

```
Client                        API Gateway + Lambda                  AWS
  │                                     │                            │
  │── POST /documents/prepare ─────────►│                            │
  │   Authorization: Bearer <JWT>        │                            │
  │                                     │ TenantMiddleware            │
  │                                     │  → resolves tenantId        │
  │                                     │ generates documentId (GUID) │
  │                                     │── PutObject pre-signed ────►│ S3
  │                                     │   URL (15 min TTL,          │
  │                                     │   PUT only, PDF only)       │
  │                                     │── writes PENDING record ───►│ DynamoDB
  │◄── { documentId, uploadUrl } ───────│                            │
  │                                     │                            │
  │── PUT {uploadUrl} (file bytes) ────────────────────────────────►│ S3
  │   (direct — bypasses Lambda)         │                            │
  │◄── 200 OK ──────────────────────────────────────────────────────│
  │                                     │                            │
  │                                     │         GuardDuty scans object (async, seconds)
  │                                     │                            │
  │── POST /documents/process ─────────►│                            │
  │   { "documentId": "...",            │ reads GuardDutyMalware     │
  │     "s3Key": "...",                 │ ScanStatus tag ───────────►│ S3
  │     "documentType": "Invoice" }     │                            │
  │                                     │  THREATS_FOUND/UNSCANNABLE │
  │                                     │    → 422 Unprocessable     │
  │                                     │  tag absent                │
  │                                     │    → 409 Conflict (retry)  │
  │                                     │  CLEAN → proceed           │
  │                                     │── ExtractTextAsync ───────►│ Textract
  │                                     │◄── raw text ───────────────│
  │                                     │── InvokeModel ────────────►│ Bedrock/Claude
  │                                     │◄── structured fields ──────│
  │                                     │── writes result ──────────►│ DynamoDB
  │                                     │── publishes event ────────►│ SQS
  │◄── 200 OK { documentId, fields } ──│                            │
```

> See [ADR-005](adrs/005-upload-strategy.md) for the upload mechanism rationale (pre-signed URL vs. backend proxy).
> See [ADR-004](adrs/004-malware-scanning.md) for the malware scanning gate logic.

---

## Step 1 — Prepare (`POST /documents/prepare`)

```
API Gateway → TenantMiddleware → DocumentEndpoints.PrepareUpload
  │
  │  Validates Cognito JWT
  │  Resolves tenantId from custom:tenantId claim
  │  Generates documentId (GUID)
  │  Constructs S3 key: {tenantId}/{year}/{month}/{documentId}.pdf
  │  Generates pre-signed PUT URL (15-minute TTL, Content-Type: application/pdf)
  │  Writes DynamoDB record: PK=TENANT#{tenantId}, SK=DOCUMENT#{documentId}, Status=PENDING
  ▼
Response: { documentId, uploadUrl }
```

---

## Step 2 — Upload (Client → S3 directly)

The client PUTs the file bytes directly to the pre-signed URL. This call goes to S3, not to API Gateway or Lambda.

```
Client PUT {uploadUrl}
  Content-Type: application/pdf
  Body: <file bytes>
  ▼
S3 accepts the object at {tenantId}/{year}/{month}/{documentId}.pdf
  ▼
GuardDuty Malware Protection scans the object (seconds)
  ▼
S3 object tagged: GuardDutyMalwareScanStatus = CLEAN | THREATS_FOUND | UNSCANNABLE
```

---

## Step 3 — Process (`POST /documents/process`)

```
API Gateway → TenantMiddleware → DocumentEndpoints.ProcessDocument
  │
  ▼
DocumentExtractionService.ExtractAsync
  │
  ├── S3:GetObjectTagging(s3Key)
  │     GuardDutyMalwareScanStatus = THREATS_FOUND → 422 Unprocessable Entity
  │     GuardDutyMalwareScanStatus = UNSCANNABLE   → 422 Unprocessable Entity
  │     tag absent                                 → 409 Conflict (scan pending, retry)
  │     GuardDutyMalwareScanStatus = CLEAN         → proceed
  │
  ├──► IOcrService.ExtractTextAsync(s3Key)
  │      │
  │      │  [Hybrid fast path — see ADR-003]
  │      ├── PdfPig: in-process extraction from PDF bytes
  │      │     ✓ text > threshold → return immediately (free, milliseconds)
  │      │     ✗ text < threshold → fall through
  │      └── TextractOcrService: DetectDocumentText / StartDocumentTextDetection
  │            → returns raw text string
  │
  └──► ISemanticAnalysisService.AnalyzeAsync(rawText, documentType)
         │
         │  BedrockSemanticAnalysisService:
         │    BuildPrompt(documentType) → type-specific extraction prompt
         │    InvokeModel (ModelId from BedrockOptions config) → Claude on Bedrock
         │    Parse JSON response → Dictionary<string, string> fields
         └── returns extracted fields
  │
  │  Updates DynamoDB record: Status=COMPLETED, Fields=..., ProcessedAt=...
  │  Publishes ExtractionCompleted event to SQS (notifier only)
  │  Assembles ExtractionResult:
  │    { DocumentId, TenantId, DocumentType, Fields, ProcessedAt }
  ▼
HTTP 200 OK
{ "documentId": "...", "tenantId": "...", "documentType": "...", "fields": { ... }, "processedAt": "..." }
```

---

## S3 Storage Convention

```
s3://<bucket>/{tenantId}/{year}/{month}/{documentId}.pdf
```

For RAG ingestion, a companion metadata file is written alongside the document:

```
s3://<bucket>/{tenantId}/{year}/{month}/{documentId}.pdf.metadata.json
```

```json
{
  "metadataAttributes": {
    "tenantId": "tenant-abc",
    "documentType": "invoice",
    "documentId": "doc-123"
  }
}
```

---

## DynamoDB Storage Convention

```
PK: TENANT#{tenantId}
SK: DOCUMENT#{documentId}
```

Document lifecycle states written to this record:

| Status | Written at |
|---|---|
| `PENDING` | `POST /documents/prepare` |
| `COMPLETED` | `POST /documents/process` — success |
| `REJECTED` | `POST /documents/process` — scan failed |

---

## RAG Ingestion Flow

After extraction, if the document needs to be indexed for semantic search:

```
1. Clean text / original PDF already in S3 (from upload step)
2. Metadata file written alongside it
3. StartIngestionJob triggered on the Knowledge Base
4. Bedrock Knowledge Bases: chunk → embed → index (OpenSearch Serverless)

At query time:
  Retrieve(query, filter: { tenantId: "..." })
  → returns relevant chunks scoped to that tenant only
  → RetrieveAndGenerate passes chunks to Claude for answer synthesis
```

See [ADR-001](adrs/001-rag-strategy.md) for the full RAG strategy rationale.

---

## Post-Extraction Notification Flow

```
DocumentExtractionService
  → publish ExtractionCompleted event to SQS
    → consumer Lambda reads from queue
    → sends notification to tenant contact
```

!!! note
    SQS is used as a **notifier only** — it does not trigger the extraction pipeline.

---

## Key Data Contracts

### `ProcessDocumentRequest`

```csharp
record ProcessDocumentRequest(
    string DocumentId,
    string S3Key,
    DocumentType DocumentType
);
```

### `ExtractionResult`

```csharp
record ExtractionResult
{
    string DocumentId
    string TenantId
    DocumentType DocumentType
    Dictionary<string, string> Fields
    DateTimeOffset ProcessedAt
}
```

### `DocumentType` (enum)

Supported values: `Invoice`, `Contract`, `Report`, `CV`.

To add a new type: add the variant to `Models/DocumentType.cs` and add a matching `case` in `BedrockSemanticAnalysisService.BuildPrompt`.

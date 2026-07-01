# Data Flow

## Full Upload and Processing Flow

The complete flow from a client uploading a document to receiving extraction results spans three steps: prepare, upload, and process.

```mermaid
sequenceDiagram
    participant C as Tenant Client
    participant GW as API Gateway + Lambda
    participant S3 as Amazon S3
    participant GD as GuardDuty
    participant TX as Textract
    participant BD as Bedrock / Claude
    participant DB as DynamoDB
    participant SQ as SQS

    C->>GW: POST /documents/prepare (JWT)
    GW->>GW: validate tenantId claim
    GW->>S3: generate pre-signed PUT URL
    GW->>DB: write PENDING record
    GW-->>C: { documentId, versionNumber, uploadUrl }

    C->>S3: PUT file (direct — bypasses Lambda)
    S3-->>C: 200 OK
    S3-)GD: trigger malware scan (async)
    GD->>S3: tag object (CLEAN / THREATS_FOUND / UNSCANNABLE)

    C->>GW: POST /documents/process
    GW->>S3: GetObjectTagging

    alt THREATS_FOUND or UNSCANNABLE
        GW-->>C: 422 Unprocessable Entity
    else tag absent — scan still running
        GW-->>C: 409 Conflict (retry with backoff)
    else CLEAN
        GW->>TX: ExtractTextAsync
        TX-->>GW: raw text
        GW->>BD: InvokeModel (Claude)
        BD-->>GW: structured fields
        GW->>GW: compute SHA-256 of PDF bytes
        GW->>DB: check SHA vs latest version
        alt SHA matches — duplicate
            GW-->>C: 200 OK { status: DUPLICATE, versionNumber }
        else new content
            GW->>DB: fetch previous version fields
            GW->>GW: compute JSON Patch diff
            GW->>DB: write COMPLETED + fields + diff
            GW->>SQ: publish ExtractionCompleted
            GW-->>C: 200 OK { documentId, versionNumber, fields, diffFromPrevious }
        end
    end
```

> See [ADR-005](adrs/005-upload-strategy.md) for the upload mechanism rationale (pre-signed URL vs. backend proxy).
> See [ADR-004](adrs/004-malware-scanning.md) for the malware scanning gate logic.

---

## Step 1 — Prepare (`POST /documents/prepare`)

```mermaid
flowchart TD
    A["POST /documents/prepare"] --> B["TenantMiddleware\nresolves tenantId from JWT"]
    B --> C{documentId\nprovided?}
    C -->|No — new document| D["Generate documentId (GUID)\nversionNumber = 1"]
    C -->|Yes — new version| E["Validate documentId belongs\nto authenticated tenant\nversionNumber = latestVersion + 1"]
    D --> F["Construct S3 key\n{tenantId}/documents/{documentId}/v{n}.pdf"]
    E --> F
    F --> G["Generate pre-signed PUT URL\n15-min TTL · PUT only · application/pdf"]
    G --> H["Write PENDING version record\nDynamoDB SK=DOCUMENT#id#VERSION#n"]
    H --> I["Return { documentId, versionNumber, uploadUrl }"]
```

---

## Step 2 — Upload (Client → S3 directly)

The client PUTs the file bytes directly to the pre-signed URL. This call goes to S3, not to API Gateway or Lambda.

```mermaid
flowchart TD
    A["Client PUT {uploadUrl}\nContent-Type: application/pdf\nBody: file bytes"] --> B["S3 stores object\n{tenantId}/{year}/{month}/{documentId}.pdf"]
    B --> C["GuardDuty Malware Protection\nscans object — typically seconds"]
    C --> D["Tag object:\nGuardDutyMalwareScanStatus\nCLEAN · THREATS_FOUND · UNSCANNABLE"]
```

---

## Step 3 — Process (`POST /documents/process`)

```mermaid
flowchart TD
    A["POST /documents/process"] --> B["S3:GetObjectTagging"]
    B --> C{GuardDutyMalwareScanStatus}

    C -->|THREATS_FOUND| D["422 Unprocessable Entity"]
    C -->|UNSCANNABLE| D
    C -->|tag absent| E["409 Conflict\nscan pending — retry with backoff"]
    C -->|CLEAN| F["Compute SHA-256\nof PDF bytes from S3"]
    F --> G{SHA matches\nlatest version?}
    G -->|Yes — duplicate| H["Write DUPLICATE record\n200 OK { status: DUPLICATE, versionNumber }"]
    G -->|No — new content| I["IOcrService.ExtractTextAsync"]

    I --> J{PdfPig result}
    J -->|"text > threshold\ndigital PDF"| K["Return raw text\nfree · milliseconds · no network call"]
    J -->|"text < threshold\nscanned document"| L["TextractOcrService\nDetectDocumentText"]
    L --> K

    K --> M["ISemanticAnalysisService.AnalyzeAsync\nBuildPrompt → InvokeModel Claude → parse JSON"]
    M --> N{Previous\nversion exists?}
    N -->|No — v1| O["Write COMPLETED record\nfields only · no diff"]
    N -->|Yes| P["Compute JSON Patch diff\nfields v_prev → fields v_new"]
    P --> Q["Write COMPLETED record\nfields + diffFromPrevious"]
    O --> R["Publish ExtractionCompleted to SQS"]
    Q --> R
    R --> S["200 OK\n{ documentId · versionNumber · fields · diffFromPrevious }"]
```

> See [ADR-003](adrs/003-ocr-strategy.md) for the OCR hybrid fast path rationale.
> See [ADR-006](adrs/006-sync-vs-async-processing.md) for the synchronous processing decision.

---

## S3 Storage Convention

```
s3://<bucket>/{tenantId}/documents/{documentId}/v{versionNumber}.pdf
```

For RAG ingestion, a companion metadata file is written alongside the document:

```
s3://<bucket>/{tenantId}/documents/{documentId}/v{versionNumber}.pdf.metadata.json
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

Two item types per document, both under the same partition key:

```
PK: TENANT#{tenantId}

SK: DOCUMENT#{documentId}                      ← group record (one per document)
SK: DOCUMENT#{documentId}#VERSION#{0001}       ← version record (one per upload)
```

SK version numbers are zero-padded to 4 digits to support chronological range queries.

**Group record** — created on first `POST /documents/prepare`, updated on each new version:

| Attribute | Description |
|---|---|
| `documentId` | Stable document identifier |
| `documentType` | Document type |
| `latestVersion` | Current highest version number |
| `createdAt` / `updatedAt` | ISO 8601 timestamps |

**Version record** — lifecycle states:

| Status | Written at |
|---|---|
| `PENDING` | `POST /documents/prepare` |
| `COMPLETED` | `POST /documents/process` — extraction succeeded |
| `DUPLICATE` | `POST /documents/process` — SHA matches previous version |
| `REJECTED` | `POST /documents/process` — scan failed (THREATS_FOUND / UNSCANNABLE) |

---

## RAG Ingestion Flow

After extraction, if the document needs to be indexed for semantic search:

```mermaid
flowchart TD
    A["Clean text / original PDF\nalready in S3 from upload step"] --> B["Write metadata file\n{documentId}.pdf.metadata.json"]
    B --> C["StartIngestionJob\nBedrock Knowledge Base"]
    C --> D["Chunk · Embed · Index\nOpenSearch Serverless"]

    E["Query time:\nRetrieve with filter tenantId"] --> F["Returns chunks scoped\nto that tenant only"]
    F --> G["RetrieveAndGenerate\npasses chunks to Claude for synthesis"]
```

See [ADR-001](adrs/001-rag-strategy.md) for the full RAG strategy rationale.

---

## Post-Extraction Notification Flow

```mermaid
flowchart LR
    A["DocumentExtractionService"] -->|"publish ExtractionCompleted"| B["Amazon SQS"]
    B --> C["Consumer Lambda\nreads from queue"]
    C --> D["Send notification\nto tenant contact"]
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

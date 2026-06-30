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
    GW-->>C: { documentId, uploadUrl }

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
        GW->>DB: update COMPLETED + fields
        GW->>SQ: publish ExtractionCompleted
        GW-->>C: 200 OK { documentId, fields }
    end
```

> See [ADR-005](adrs/005-upload-strategy.md) for the upload mechanism rationale (pre-signed URL vs. backend proxy).
> See [ADR-004](adrs/004-malware-scanning.md) for the malware scanning gate logic.

---

## Step 1 — Prepare (`POST /documents/prepare`)

```mermaid
flowchart TD
    A["POST /documents/prepare"] --> B["TenantMiddleware\nresolves tenantId from JWT"]
    B --> C["Generate documentId\nGUID"]
    C --> D["Construct S3 key\n{tenantId}/{year}/{month}/{documentId}.pdf"]
    D --> E["Generate pre-signed PUT URL\n15-min TTL · PUT only · application/pdf"]
    E --> F["Write PENDING record\nDynamoDB PK=TENANT#tenantId SK=DOCUMENT#documentId"]
    F --> G["Return { documentId, uploadUrl }"]
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
    C -->|CLEAN| F["IOcrService.ExtractTextAsync"]

    F --> G{PdfPig result}
    G -->|"text > threshold\ndigital PDF"| H["Return raw text\nfree · milliseconds · no network call"]
    G -->|"text < threshold\nscanned document"| I["TextractOcrService\nDetectDocumentText"]
    I --> H

    H --> J["ISemanticAnalysisService.AnalyzeAsync\nBuildPrompt → InvokeModel Claude → parse JSON"]
    J --> K["Update DynamoDB\nStatus = COMPLETED · Fields · ProcessedAt"]
    K --> L["Publish ExtractionCompleted to SQS"]
    L --> M["200 OK\n{ documentId · tenantId · documentType · fields · processedAt }"]
```

> See [ADR-003](adrs/003-ocr-strategy.md) for the OCR hybrid fast path rationale.
> See [ADR-006](adrs/006-sync-vs-async-processing.md) for the synchronous processing decision.

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

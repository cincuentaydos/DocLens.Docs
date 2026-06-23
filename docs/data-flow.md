# Data Flow

## Document Processing Flow

```
Client
  │
  │  POST /documents/process
  │  Authorization: Bearer <Cognito JWT>
  │  { "documentId": "...", "s3Key": "...", "documentType": "Invoice" }
  ▼
API Gateway (HTTP API v2)
  │
  │  Validates JWT signature via Cognito
  │  Forwards request + decoded claims to Lambda
  ▼
TenantMiddleware
  │
  │  Reads custom:tenantId claim from JWT
  │  Returns 401 if claim is missing or empty
  │  Sets TenantContext.TenantId for the request lifetime
  ▼
DocumentEndpoints.ProcessDocument
  │
  │  Deserialises ProcessDocumentRequest
  │  Delegates to IDocumentExtractionService
  ▼
DocumentExtractionService.ExtractAsync
  │
  ├──► IOcrService.ExtractTextAsync(s3Key)
  │      │
  │      │  [Hybrid fast path]
  │      ├── PdfPig: try in-process text extraction from PDF bytes
  │      │     ✓ If text > threshold → return text immediately
  │      │     ✗ If text < threshold → fall through to Textract
  │      └── TextractOcrService: DetectDocumentText / StartDocumentTextDetection
  │            → Returns raw text string
  │
  └──► ISemanticAnalysisService.AnalyzeAsync(rawText, documentType)
         │
         │  BedrockSemanticAnalysisService:
         │    BuildPrompt(documentType) → type-specific extraction prompt
         │    InvokeModel → Claude on Amazon Bedrock
         │    Parse JSON response → Dictionary<string, string> fields
         └── Returns extracted fields
  │
  │  Assembles ExtractionResult:
  │    { DocumentId, TenantId, DocumentType, Fields, ProcessedAt }
  ▼
API Gateway → Client
  HTTP 200 OK
  { "documentId": "...", "tenantId": "...", "fields": { ... }, "processedAt": "..." }
```

## S3 Storage Convention

Documents uploaded by a tenant are stored under a path that encodes tenant, date, and document identity:

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

## DynamoDB Storage Convention

Extraction results are persisted with a partition key that scopes data per tenant:

```
PK: TENANT#{tenantId}
SK: DOCUMENT#{documentId}
```

## RAG Ingestion Flow

After extraction, if the document needs to be indexed for semantic search / RAG queries:

```
1. Clean text / original PDF written to S3 (path above)
2. Metadata file written alongside it
3. StartIngestionJob triggered on the Knowledge Base
4. Bedrock Knowledge Bases: chunk → embed → index (OpenSearch Serverless)

At query time:
  Retrieve(query, filter: { tenantId: "..." })
  → Returns relevant chunks scoped to that tenant only
  → RetrieveAndGenerate passes chunks to Claude for answer synthesis
```

## Post-Extraction Notification Flow

After a successful extraction, an event is published to SQS for downstream consumers:

```
DocumentExtractionService
  → Publish ExtractionCompleted event to SQS
    → Consumer (e.g., email Lambda) reads from queue
    → Sends notification to tenant contact
```

!!! note
    SQS is used as a **notifier only** — it does not trigger the extraction pipeline itself.

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

# Architecture

## Tech Stack

| Layer | Technology |
|---|---|
| Compute | AWS Lambda (.NET 10) |
| API Entry | Amazon API Gateway (HTTP API v2) |
| Identity | Amazon Cognito (JWT with `custom:tenantId` claim) |
| OCR | Amazon Textract + PdfPig (hybrid fast path) |
| AI Analysis | Amazon Bedrock — Claude via `InvokeModel` |
| RAG | Amazon Bedrock Knowledge Bases (chunking, embedding, retrieval) |
| Operational DB | Amazon DynamoDB |
| Document Storage | Amazon S3 |
| Notifications | Amazon SNS / SQS (post-extraction notifications) |
| Security | Amazon GuardDuty Malware Protection for S3 (upload scanning) |
| Observability | Amazon CloudWatch + AWS X-Ray + AWS Lambda Powertools |
| IaC | Terraform (HCL) — primary; AWS CDK scaffolding present but being phased out |
| Frontend | React 19, TypeScript, Vite, Feature-Sliced Design |
| Frontend IaC | Terraform (environments: dev / staging / production) |

## Component Map

```mermaid
graph TD
    Client["Tenant Client"]

    Client -->|"HTTPS — API calls"| APIGW["Amazon API Gateway\nHTTP API v2"]
    Client -->|"PUT — pre-signed URL\ndirect upload, bypasses Lambda"| S3

    APIGW -->|"JWT validated via Cognito"| Lambda["AWS Lambda .NET 10\nTenantMiddleware · DocumentEndpoints\nExtractionService · OcrService · SemanticService"]

    Lambda -->|"OCR"| Textract["Amazon Textract"]
    Lambda -->|"semantic analysis"| Bedrock["Amazon Bedrock\nClaude via InvokeModel"]
    Bedrock -.->|"RAG"| KB["Bedrock Knowledge Bases\nchunking · embedding · retrieval"]

    Lambda -->|"reads scan tag · writes result"| S3["Amazon S3\n{tenantId}/{year}/{month}/{documentId}.pdf"]
    S3 -->|"auto-scan on upload"| GD["GuardDuty\nMalware Protection for S3"]

    Lambda --> DDB["Amazon DynamoDB\nPK: TENANT#tenantId · SK: DOCUMENT#documentId"]
    Lambda -->|"notifier only"| SQS["Amazon SQS\npost-extraction events"]
```

## Multi-Tenancy Model

DocLens uses a **logical silo** approach: all tenants share the same infrastructure, but data is partitioned by `tenantId` at every layer.

| Layer | Isolation key |
|---|---|
| API / Auth | `custom:tenantId` claim in Cognito JWT |
| Middleware | `TenantMiddleware` resolves `TenantId` before any handler runs |
| S3 | Key prefix: `{tenantId}/{year}/{month}/{documentId}.pdf` |
| DynamoDB | Partition key: `TENANT#{tenantId}` |
| Bedrock Knowledge Bases | Metadata filter: `{ "tenantId": "..." }` on every `Retrieve` call |
| Logs | Structured log field `TenantId` on every log entry |

!!! danger "Critical rule"
    The `tenantId` filter on Knowledge Bases **must never be optional** — it is the only barrier preventing cross-tenant retrieval.

## Design Principles

- **Inside-out design:** core extraction logic is built first; infrastructure wiring is added second.
- **Interface + implementation pairs:** every service has an interface (`IOcrService`, `ISemanticAnalysisService`, `IDocumentExtractionService`) to enable mocking in tests and future substitution.
- **Scoped services, singleton AWS clients:** services are registered as `Scoped`; AWS SDK clients are `Singleton` via `AddAWSService<T>`.
- **Synchronous extraction (current):** Textract + Bedrock are called inline within the Lambda invocation. Async job queue will be added when latency or throughput require it.
- **SQS as notifier only:** not used as a processing trigger — only for post-extraction notifications (e.g., email).

## OCR Strategy — Hybrid Fast Path

See [ADR-003](adrs/003-ocr-strategy.md) for the full analysis.

1. **PdfPig first** — attempts in-process extraction from the PDF byte stream (free, milliseconds, no network call).
2. **Evaluate result** — if extracted text is below a minimum threshold (~50 characters), the document is treated as image-based.
3. **Textract fallback** — only invoked when PdfPig yields insufficient text (scanned documents, photos of documents).

## RAG Strategy — Bedrock Knowledge Bases

See [ADR-001](adrs/001-rag-strategy.md) for the full analysis.

DocLens owns: document intake, OCR, S3 layout, metadata authoring, ingestion job triggering.
Bedrock Knowledge Bases owns: chunking (semantic strategy), embedding, vector store, index lifecycle.

Tenant isolation at the RAG layer is enforced via S3 metadata files and a mandatory `tenantId` filter on every `Retrieve` / `RetrieveAndGenerate` call.

## Upload Strategy — Pre-signed URL

See [ADR-005](adrs/005-upload-strategy.md) for the full analysis.

Clients upload documents directly to S3 using a short-lived pre-signed PUT URL, bypassing Lambda entirely. This avoids the 10 MB API Gateway payload limit and eliminates unnecessary data transfer through the compute layer.

1. Client calls `POST /documents/prepare` — receives `{ documentId, uploadUrl }`.
2. Client PUTs the file directly to S3 using `uploadUrl` (15-minute TTL, `application/pdf` only).
3. GuardDuty scans the object and tags it with the scan result.
4. Client calls `POST /documents/process` — Lambda reads the scan tag before invoking Textract or Bedrock.

## Malware Scanning — GuardDuty

See [ADR-004](adrs/004-malware-scanning.md) for the full analysis.

GuardDuty Malware Protection for S3 scans every uploaded object automatically. The processing Lambda gates extraction on the `GuardDutyMalwareScanStatus` tag: `CLEAN` proceeds, `THREATS_FOUND` and `UNSCANNABLE` return `422`, and an absent tag (scan still running) returns `409`.

## IaC Strategy

See [ADR-002](adrs/002-iac-strategy.md) for the full rationale.

- **Terraform (HCL)** is the primary IaC tool for all DocLens infrastructure.
- Terraform state: S3 backend + DynamoDB lock table.
- Terraform code lives in `infra/terraform/` (Lambda project) and `infra/` (Web template).
- AWS CDK scaffolding (`infra/src/DocLens.Infra/`) exists from early exploration and is being removed as Terraform coverage grows.

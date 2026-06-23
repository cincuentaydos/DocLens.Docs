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
| Observability | Amazon CloudWatch + AWS X-Ray + AWS Lambda Powertools |
| IaC | AWS CDK (C#) — primary; Terraform scaffolding also present |
| Frontend | React 19, TypeScript, Vite, Feature-Sliced Design |
| Frontend IaC | Terraform (environments: dev / staging / production) |

## Component Map

```
                    ┌───────────────────────────────────────────┐
                    │              Tenant (Client)               │
                    └────────────────────┬──────────────────────┘
                                         │ HTTPS
                    ┌────────────────────▼──────────────────────┐
                    │         Amazon API Gateway (HTTP v2)       │
                    └────────────────────┬──────────────────────┘
                                         │ JWT (Cognito)
                    ┌────────────────────▼──────────────────────┐
                    │            AWS Lambda (.NET 10)            │
                    │                                            │
                    │  TenantMiddleware ──► DocumentEndpoints    │
                    │                          │                 │
                    │              DocumentExtractionService     │
                    │               ┌──────────┴──────────┐     │
                    │        OcrService          SemanticService │
                    │       (Textract)            (Bedrock)      │
                    └───────┬───────────────────────┬───────────┘
                            │                       │
          ┌─────────────────▼───┐      ┌────────────▼────────────┐
          │    Amazon Textract   │      │    Amazon Bedrock        │
          │    (OCR / text       │      │    (Claude — field       │
          │     extraction)      │      │     extraction)          │
          └─────────────────────┘      └─────────────────────────┘
                                                    │
                                       ┌────────────▼────────────┐
                                       │  Bedrock Knowledge Bases │
                                       │  (RAG: chunking,         │
                                       │   embedding, retrieval)  │
                                       └─────────────────────────┘
          ┌──────────────────────────────────────────────────────┐
          │  Amazon S3          Amazon DynamoDB     Amazon SQS   │
          │  (document store)   (extraction results) (notifier)  │
          └──────────────────────────────────────────────────────┘
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

## IaC Strategy

See [ADR-002](adrs/002-iac-strategy.md) for the full analysis (CDK vs Terraform — decision pending).

- AWS CDK (C#) is the current primary IaC tooling for the Lambda backend.
- Terraform scaffolding exists both in `infra/terraform/` (Lambda project) and `infra/` (Web template).
- Manual `dotnet lambda deploy-function` is used during early development.

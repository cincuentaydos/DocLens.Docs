# ADR-001 — RAG Strategy

**Status:** Accepted (2026-06-10)

---

## Decision

Use **Amazon Bedrock Knowledge Bases** as the RAG infrastructure layer.

DocLens retains ownership of document intake, OCR pipeline, S3 layout, metadata authoring, and ingestion job triggering. Bedrock Knowledge Bases owns chunking, embedding, vector store, and index lifecycle.

## Why Bedrock Knowledge Bases

| Factor | Decision |
|---|---|
| Time to value | Knowledge Bases ships in days, not weeks — no cluster to provision |
| Separation of concerns | DocLens focuses on document quality; AWS handles retrieval infrastructure |
| Chunking strategy | Semantic chunking (NLP-based) is the default for DocLens document types |
| Tenant isolation | Logical silo via metadata filter `{ "tenantId": "..." }` on every `Retrieve` call |
| Migration path | `IRetrievalService` interface hides the implementation — can be replaced with OpenSearch if needed |

## Constraints to Watch

!!! danger "Tenant isolation"
    The `tenantId` filter on every retrieval call must never be optional — it is the only cross-tenant barrier.

- Chunking is not fully controlled (cannot use Textract block boundaries directly).
- Embedding model limited to Bedrock's catalogue (Titan Embeddings V2, Cohere Embed v3).
- AWS lock-in at the RAG layer.

## Revisit Triggers

- Physical data isolation required by contract or regulation.
- Retrieval quality is insufficient for tenant use cases.
- A non-Bedrock embedding model is required.
- Pricing at scale becomes a concern.

---

## Open Questions

- **Ingestion trigger — not yet defined.** This ADR says DocLens owns "ingestion job triggering," but the mechanism itself isn't specified anywhere. `StartIngestionJob` is a distinct, genuinely asynchronous AWS-managed job — separate from the `POST /documents/process` pipeline in [ADR-006](006-sync-vs-async-processing.md), which only covers OCR (Textract) + field extraction (Bedrock `InvokeModel`). Open points to resolve:
    - What triggers `StartIngestionJob` — the same Worker Lambda, right after it writes the `COMPLETED` record? A separate consumer of the `ExtractionCompleted` message? A scheduled/batch job?
    - Note: the `ExtractionCompleted` SQS queue is documented (see `data-flow.md`) as a **notifier only** — it does not trigger the extraction pipeline. If it ends up also triggering RAG ingestion, that changes its role and should be called out explicitly.
    - Does ingestion happen per-version (every uploaded version gets (re-)indexed) or only for the latest version of a document?
    - What happens to the previous version's index entries when a new version is ingested — replaced, or does the KB retain both?

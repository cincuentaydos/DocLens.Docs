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

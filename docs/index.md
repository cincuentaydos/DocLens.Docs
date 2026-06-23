# DocLens

**Multi-tenant intelligent document extraction platform on AWS Serverless.**

Companies upload documents — invoices, contracts, reports, CVs — and receive structured data extracted automatically through a pipeline of OCR and semantic AI analysis.

---

## Repositories

| Repository | Description |
|---|---|
| [`DocLens.Lambda.Template`](https://github.com/cincuentaydos/DocLens.Lambda.Template) | Backend core — document processing API (.NET 10, Lambda) |
| [`DocLens.Web.Template`](https://github.com/cincuentaydos/DocLens.Web.Template) | React 19 frontend template (Feature-Sliced Design) |
| [`DocLens.Skills`](https://github.com/cincuentaydos/DocLens.Skills) | GitHub Copilot / Claude Code plugin marketplace |

## Quick Navigation

<div class="grid cards" markdown>

- :material-file-document-outline: **[Overview](overview.md)**

    What DocLens is, its purpose, and how it is structured.

- :material-floor-plan: **[Architecture](architecture.md)**

    AWS services, component map, multi-tenancy model, and design principles.

- :material-chart-timeline-variant: **[Data Flow](data-flow.md)**

    End-to-end request flow from document upload to extraction result.

- :material-vote: **[ADRs](adrs/001-rag-strategy.md)**

    Architecture Decision Records — RAG, IaC, and OCR strategy.

</div>

## Primary Region

| Region | Role |
|---|---|
| `eu-west-1` (Ireland) | Primary |
| `eu-west-2` (London) | Failover consideration |

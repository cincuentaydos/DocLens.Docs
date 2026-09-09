---
template: home.html
hide:
  - toc
---

## Repositorios

| Repositorio | Descripción |
|---|---|
| [`DocLens.Lambda.Template`](https://github.com/cincuentaydos/DocLens.Lambda.Template) | Núcleo del backend — API de gestión de casos legales y procesamiento de documentos (.NET 10, Lambda) |
| [`DocLens.Web.Template`](https://github.com/cincuentaydos/DocLens.Web.Template) | Plantilla de frontend en React 19 (Feature-Sliced Design) |
| [`DocLens.Skills`](https://github.com/cincuentaydos/DocLens.Skills) | Marketplace de plugins para GitHub Copilot / Claude Code |

## Navegación Rápida

<div class="grid cards" markdown>

- :material-file-document-outline: **[Resumen](overview.md)**

    Qué es DocLens, su propósito, y cómo está estructurado.

- :material-floor-plan: **[Arquitectura](architecture.md)**

    Servicios de AWS, mapa de componentes, modelo de multi-tenancy y principios de diseño.

- :material-chart-timeline-variant: **[Flujo de Datos](data-flow.md)**

    Flujo completo, desde la subida de un documento hasta el resultado de la extracción y la consulta con IA.

- :material-vote: **[ADRs](adrs/001-rag-strategy.md)**

    Registros de Decisiones de Arquitectura — RAG, IaC, almacenamiento de datos, red, cómputo, disparo de procesamiento, IA y recuperación ante desastres.

</div>

## Región Principal

| Región | Rol |
|---|---|
| `eu-west-1` (Irlanda) | Primaria — única región activa en V1 (ver [ADR-013](adrs/013-disaster-recovery-strategy.md)) |

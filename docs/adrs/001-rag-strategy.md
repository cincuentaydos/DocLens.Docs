# ADR-001 — Estrategia de RAG

**Estado:** Aceptada (2026-06-10) — motor de vector store actualizado, ver nota abajo.

---

## Decisión

Usar **Amazon Bedrock Knowledge Bases** como capa de infraestructura de RAG.

DocLens mantiene la propiedad de la ingesta de documentos, el pipeline de OCR, la organización en S3, la generación de metadatos y el disparo de los jobs de ingesta. Bedrock Knowledge Bases se encarga del chunking, el embedding, el vector store y el ciclo de vida del índice.

**Motor de vector store:** ver [ADR-008](008-data-storage-strategy.md) — el vector store usado por Bedrock Knowledge Bases es **Aurora PostgreSQL Serverless v2 con `pgvector`** (esquema `kb`), no un motor gestionado aparte como OpenSearch Serverless.

## Por qué Bedrock Knowledge Bases

| Factor | Decisión |
|---|---|
| Tiempo de valor | Knowledge Bases se pone en marcha en días, no semanas — sin clúster que aprovisionar |
| Separación de responsabilidades | DocLens se enfoca en la calidad del documento; AWS gestiona la infraestructura de retrieval |
| Estrategia de chunking | El chunking semántico (basado en NLP) es el predeterminado para los tipos de documento de DocLens |
| Aislamiento por tenant | Silo lógico vía filtro de metadatos `{ "empresa_id": "..." }` en cada llamada `Retrieve` |
| Vector store | Aurora PostgreSQL + `pgvector` (ver [ADR-008](008-data-storage-strategy.md)) — reutiliza el mismo motor que los datos operativos |
| Ruta de migración | Una interfaz `IRetrievalService` oculta la implementación — puede reemplazarse si es necesario |

## Restricciones a vigilar

!!! danger "Aislamiento por tenant"
    El filtro `empresa_id` en cada llamada de retrieval nunca debe ser opcional — es la única barrera contra el acceso cruzado entre tenants.

- El chunking no es totalmente controlable (no se pueden usar directamente los límites de bloque de Textract).
- El modelo de embedding está limitado al catálogo de Bedrock (Titan Embeddings V2, Cohere Embed v3).
- Dependencia de AWS en la capa de RAG.

## Disparadores de revisión

- Se requiere aislamiento físico de datos por contrato o regulación.
- La calidad de retrieval resulta insuficiente para los casos de uso de los tenants.
- Se requiere un modelo de embedding fuera del catálogo de Bedrock.
- El costo a escala se vuelve una preocupación.

---

## Preguntas abiertas

- **Disparador de ingesta — aún no definido.** Esta ADR dice que DocLens es dueño del "disparo del job de ingesta", pero el mecanismo en sí no está especificado en ningún lugar. `StartIngestionJob` es un job genuinamente asíncrono y gestionado por AWS — distinto del pipeline de `POST /documents/process` que existía en la versión anterior de [ADR-006](006-sync-vs-async-processing.md) (ahora superada por [ADR-011](011-document-processing-trigger.md)), el cual solo cubre OCR (Textract) + extracción de campos (Bedrock `InvokeModel`). Puntos abiertos por resolver:
    - ¿Qué dispara `StartIngestionJob` — la misma Lambda Processor, justo después de escribir el registro `COMPLETED`? ¿Un consumidor separado del evento de finalización? ¿Un job programado/por lotes?
    - ¿La ingesta ocurre por versión (cada versión subida se (re)indexa) o solo para la última versión de un documento?
    - ¿Qué ocurre con las entradas de índice de la versión anterior cuando se ingesta una nueva versión — se reemplazan, o la KB conserva ambas?

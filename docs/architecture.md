# Arquitectura

## Stack Tecnológico

| Capa | Tecnología |
|---|---|
| DNS (Edge / Perímetro) | Amazon Route 53 |
| WAF (Edge / Perímetro) | AWS WAF + AWS Shield Standard + AWS Certificate Manager, adjuntos a CloudFront (ver [ADR-009](adrs/009-edge-network-strategy.md)) |
| CDN / entrada de la app | Amazon CloudFront — único dominio para assets estáticos y llamadas de API |
| Identidad | Amazon Cognito (JWT, expuesto directo a internet) |
| Frontend (hosting) | Amazon S3 (bucket privado, servido vía CloudFront) |
| Entrada de API | Amazon API Gateway |
| Cómputo | AWS Lambda (.NET 10) (ver [ADR-010](adrs/010-compute-strategy.md)) |
| Extracción de texto | Amazon Textract + PdfPig (ruta híbrida) (ver [ADR-003](adrs/003-ocr-strategy.md)) |
| Análisis de IA | Amazon Bedrock — Claude |
| RAG | Amazon Bedrock Knowledge Bases (chunking, embedding, retrieval) (ver [ADR-001](adrs/001-rag-strategy.md)) |
| Base de datos operativa + vector store | Amazon Aurora PostgreSQL Serverless v2 + `pgvector` (ver [ADR-008](adrs/008-data-storage-strategy.md)) |
| Almacenamiento de documentos | Amazon S3 |
| Disparo de procesamiento | GuardDuty Malware Protection + EventBridge (ver [ADR-011](adrs/011-document-processing-trigger.md)) |
| Orquestación de flujos asíncronos | AWS Step Functions — 3 state machines (Extracción, Revisión humana, Sync KB) (ver [ADR-011](adrs/011-document-processing-trigger.md)) |
| Desacople de carga | Amazon SQS + DLQ (colas puntuales — ver nota sobre `RetryPolicy`/`DeadLetterConfig` de EventBridge en [ADR-011](adrs/011-document-processing-trigger.md)) |
| Seguridad de subidas | Amazon GuardDuty Malware Protection for S3 (ver [ADR-004](adrs/004-malware-scanning.md)) |
| Observabilidad | Amazon CloudWatch + AWS X-Ray |
| IaC | Terraform (HCL) (ver [ADR-002](adrs/002-iac-strategy.md)) |
| CI/CD | GitHub Actions + OIDC |
| Frontend | React 19, TypeScript, Vite, Feature-Sliced Design |

## Mapa de Componentes

Diagrama completo (click para ampliar) — incluye el flujo de subida/procesamiento de documentos y el flujo de chat, con el límite de la cuenta AWS y el código de colores por flujo (`fluxo upload doc` / `fluxo chat` / `fluxo app`):

![Diagrama completo del pipeline de documentos y del chat](assets/document-pipeline-diagram.svg)

El diagrama Mermaid abajo es un resumen simplificado en texto del mismo sistema, útil para diffs de Git y lectura rápida — el SVG de arriba es la referencia completa y detallada.

```mermaid
graph TD
    Client["Cliente / Usuario del despacho"]

    subgraph AWS["Cuenta AWS — eu-west-1, región única (ADR-013)"]
        subgraph Edge["Edge / Perímetro"]
            R53["Amazon Route 53"]
            WAF["AWS WAF\n+ Shield Standard + ACM"]
        end

        R53 --> CF["Amazon CloudFront"]
        WAF -.->|"protege"| CF
        CF -->|"assets estáticos"| S3Front["S3 — Frontend estático"]
        CF -->|"HTTPS — llamadas de API"| APIGW["Amazon API Gateway"]

        Cognito["Amazon Cognito\nemite JWT"]
        APIGW -.->|"valida JWT"| Cognito

        APIGW --> Lambda["doclens-api\nAWS Lambda .NET 10\nrutas: /prepare, /review, /reprocess, ..."]

        Lambda --> Aurora[("Aurora PostgreSQL\nServerless v2 + pgvector\nesquema public + kb")]
        Lambda -->|"consulta autorizada"| Bedrock["Amazon Bedrock\nClaude"]
        Bedrock -.->|"RAG"| KB["Bedrock Knowledge Bases\nchunking · embedding · retrieval"]
        KB -.-> Aurora

        Lambda -.->|"URL prefirmada"| S3Docs
        Client -->|"PUT — URL prefirmada\nsubida directa, evita Lambda"| S3Docs["Amazon S3 — Documentos\n{tenantId}/documents/{documentId}/v{n}.{ext}"]
        S3Docs -->|"escaneo automático al subir"| GD["GuardDuty\nMalware Protection for S3"]
        GD -.->|"THREATS_FOUND / UNSCANNABLE"| Rejected[("Aurora\nREJECTED")]
        GD -->|"resultado limpio"| EB["Amazon EventBridge"]

        EB -->|"StartExecution"| SFExtract["Step Functions\nExtracción\ndetecta formato · OCR · clasifica · analiza · redacta PII"]
        SFExtract --> Textract["Amazon Textract\n.waitForTaskToken"]
        SFExtract -->|"placeholder → valor real"| PIIMap[("Aurora\npii_mappings")]
        SFExtract --> S3KB[("S3 — KB Source\nbucket separado — ver ADR-014")]
        SFExtract -->|"PENDING_REVIEW"| Aurora
        SFExtract -.->|"evento: DocumentExtracted"| EB

        EB -->|"StartExecution"| SFReview["Step Functions\nRevisión humana\n.waitForTaskToken — gate bloqueante"]
        Lambda -.->|"POST .../review\nSendTaskSuccess/Failure"| SFReview
        SFReview -->|"APPROVED / NEEDS_CORRECTION"| Aurora
        SFReview -.->|"evento: DocumentApproved"| EB
        Aurora -.->|"NEEDS_CORRECTION →\nPOST .../reprocess (manual)"| Lambda

        EB -->|"StartExecution"| SFSync["Step Functions\nSync KB\nStartIngestionJob"]
        S3KB -.->|"lee — solo texto redactado"| SFSync
        SFSync -->|"INDEXED"| Aurora
        SFSync --> KB

        subgraph Obs["Observability"]
            CW["CloudWatch"]
            XRay["X-Ray"]
        end
        Lambda -.-> Obs
        SFExtract -.-> Obs
        SFReview -.-> Obs
        SFSync -.-> Obs
    end

    Client -->|"HTTPS"| CF
    Client -->|"login"| Cognito
```

Tres [AWS Step Functions](https://docs.aws.amazon.com/step-functions/) state machines — Extracción, Revisión humana (gate bloqueante) y Sync con el Knowledge Base — encadenadas por eventos propios en EventBridge, no una única Lambda Processor. Detalle completo en [ADR-011](adrs/011-document-processing-trigger.md).

## Modelo de Multi-Tenancy

DocLens sigue siendo una plataforma **SaaS multi-tenant**: cada despacho de abogados (empresa cliente de DocLens) es un tenant, con datos aislados a nivel lógico. Dentro de cada tenant vive el dominio del negocio del despacho — Clientes, Procesos, Documentos y Usuarios.

```
Tenant (empresa / despacho)
  ├── Usuarios internos (admin / no-admin)
  ├── Clientes
  │     └── Usuarios de cliente (consultan sus propios procesos)
  └── Procesos
        ├── Documentos (con versiones)
        └── Permisos (qué usuario accede a qué proceso)
```

| Capa | Clave de aislamiento |
|---|---|
| API / Auth | Claim `custom:tenantId` en el JWT de Cognito |
| Middleware | Resuelve el tenant antes de que corra cualquier handler |
| S3 | Prefijo de clave: `{tenantId}/documents/{documentId}/v{versionNumber}.{ext}` |
| Aurora | Columna `empresa_id` en cada tabla (ver [ADR-008](adrs/008-data-storage-strategy.md)) |
| Bedrock Knowledge Bases | Filtro de metadatos: `{ "empresa_id": "...", "proceso_id": "..." }` en cada llamada `Retrieve` |
| Logs | Campo estructurado `empresa_id` en cada entrada |

!!! danger "Regla crítica"
    Los filtros `empresa_id` **y** `proceso_id` en Knowledge Bases **nunca deben ser opcionales** — `empresa_id` es la barrera entre despachos (tenants); `proceso_id` es la barrera entre casos de un mismo despacho (ver [ADR-001](adrs/001-rag-strategy.md)).

Dentro de un mismo tenant, los **permisos por proceso** determinan qué usuario interno o de cliente puede ver qué caso — ver la tabla `permisos` en [ADR-008](adrs/008-data-storage-strategy.md) y el endpoint de auditoría de accesos en `api-reference.md`.

## Principios de Diseño

- **Procesamiento asíncrono dirigido por eventos:** el procesamiento de documentos se dispara cuando GuardDuty confirma que el archivo está limpio — no por una llamada explícita del cliente (ver [ADR-011](adrs/011-document-processing-trigger.md)).
- **Sin agente autónomo en V1:** el flujo de IA es explícito y orquestado por el backend — autorizar, recuperar de la base de conocimiento, generar con Claude, devolver con fuentes (ver [ADR-012](adrs/012-no-autonomous-agent.md)).
- **Interfaz + implementación:** cada servicio tiene una interfaz para permitir mocking en pruebas y sustitución futura.
- **Servicios scoped, clientes de AWS singleton:** los servicios se registran como `Scoped`; los clientes del SDK de AWS son `Singleton` vía `AddAWSService<T>`.
- **Cada espera asíncrona es un `waitForTaskToken`, no una recoordinación manual:** Textract, la aprobación humana y la ingesta en Bedrock KB comparten el mismo mecanismo nativo de Step Functions en vez de tres implementaciones distintas de "guardar estado y esperar" (ver [ADR-011](adrs/011-document-processing-trigger.md)).
- **El Knowledge Base nunca almacena PII en texto plano:** es una propiedad estructural del pipeline de Extracción, no una convención de código — el mapeo reversible vive aparte, en Aurora (ver [ADR-014](adrs/014-pii-redaction.md)).

## Estrategia de OCR — Ruta Híbrida

Ver [ADR-003](adrs/003-ocr-strategy.md) para el análisis completo.

1. **PdfPig primero** — intenta la extracción en proceso desde el flujo de bytes del PDF (gratis, milisegundos, sin llamada de red).
2. **Evaluar el resultado** — si el texto extraído está bajo un umbral mínimo (~50 caracteres), el documento se trata como basado en imagen.
3. **Textract como respaldo** — solo se invoca cuando PdfPig produce texto insuficiente.

## Estrategia de RAG — Bedrock Knowledge Bases

Ver [ADR-001](adrs/001-rag-strategy.md) para el análisis completo.

DocLens es dueño de: ingesta de documentos, OCR, organización en S3, generación de metadatos, disparo del job de ingesta.
Bedrock Knowledge Bases es dueño de: chunking, embedding, y ciclo de vida del índice, usando Aurora + `pgvector` como vector store (ver [ADR-008](adrs/008-data-storage-strategy.md)).

## Estrategia de Subida — URL Prefirmada

Ver [ADR-005](adrs/005-upload-strategy.md) para el análisis completo.

Los clientes suben documentos directamente a S3 usando una URL PUT prefirmada de corta duración, evitando Lambda por completo.

1. El cliente llama a `POST /documents/prepare` — recibe `{ documentId, versionNumber, uploadUrl }`.
2. El cliente hace PUT del archivo directamente a S3 usando `uploadUrl` (TTL de 15 minutos).
3. GuardDuty escanea el objeto automáticamente.
4. Si el resultado es limpio, una regla de EventBridge dispara la state machine de Extracción — sin llamada adicional del cliente (ver [ADR-011](adrs/011-document-processing-trigger.md)).
5. El cliente sondea `GET /documents/{documentId}/versions/{versionNumber}` hasta que el estado llegue a `INDEXED` (pasando por `PENDING_REVIEW` y `APPROVED`, o desviándose a `NEEDS_CORRECTION`/`REJECTED`).

## Escaneo de Malware — GuardDuty + EventBridge

Ver [ADR-004](adrs/004-malware-scanning.md) y [ADR-011](adrs/011-document-processing-trigger.md) para el análisis completo.

GuardDuty Malware Protection for S3 escanea automáticamente cada objeto subido. El resultado del escaneo se publica como evento en EventBridge; una regla filtra y dispara la state machine de Extracción únicamente para los resultados `CLEAN`. Un resultado `THREATS_FOUND` o `UNSCANNABLE` nunca llega a esa state machine — el registro correspondiente se marca `REJECTED` por una vía separada (ver pregunta abierta en [ADR-011](adrs/011-document-processing-trigger.md)).

## Versionado de Documentos

Ver [ADR-007](adrs/007-document-versioning.md) para el análisis completo.

Un `documentId` es el identificador estable de un documento a través de todas sus versiones. Cada subida crea un `versionNumber` secuencial bajo el mismo `documentId`. La state machine de Extracción calcula SHA-256 tras el escaneo de GuardDuty — contenido idéntico se marca como `DUPLICATE` sin re-extraer. Tras una extracción exitosa, se calcula y almacena un diff JSON Patch (RFC 6902) contra los campos de la versión anterior, junto con los campos completos, en Aurora.

## Base de Datos y Vector Store — Aurora + pgvector

Ver [ADR-008](adrs/008-data-storage-strategy.md) para el esquema completo.

Un único motor, Aurora PostgreSQL Serverless v2, almacena tanto los datos operativos (empresas, usuarios, clientes, procesos, documentos, permisos, auditoría) como el vector store de Bedrock Knowledge Bases (esquema `kb` dedicado, vía `pgvector`).

## IA / RAG — Flujo Explícito sin Agente

Ver [ADR-012](adrs/012-no-autonomous-agent.md) para el análisis completo.

```mermaid
flowchart LR
    A["Consulta del usuario"] --> B["Backend valida permiso\nsobre el proceso"]
    B --> C["Retrieve en Bedrock KB\nfiltrado por empresa_id + proceso_id\ncontenido redactado, con placeholders"]
    C --> D["Claude genera\nrespuesta o borrador\n(con placeholders)"]
    D --> E["Sustitución inversa vía pii_mappings\n(ver ADR-014)"]
    E --> F["Respuesta + fuentes citadas\ncon datos reales"]
```

No existe un servicio de "agente" independiente — todo el código de orquestación vive en el backend.

## Recuperación ante Desastres

Ver [ADR-013](adrs/013-disaster-recovery-strategy.md) para el análisis completo.

V1 opera en una única región (`eu-west-1`), sin despliegue activo multi-región. La recuperación se apoya en versionado/cifrado de S3, backups automáticos + Point-in-Time Recovery de Aurora, DLQ para trabajos no procesables, observabilidad vía CloudWatch, e infraestructura completamente reproducible mediante Terraform.

## Estrategia de Edge / Red

Ver [ADR-009](adrs/009-edge-network-strategy.md) para el análisis completo.

La capa de entrada es 100% nativa de AWS — Route 53, CloudFront, WAF y Shield Standard — sin un proveedor de borde externo como Cloudflare.

## Estrategia de Cómputo

Ver [ADR-010](adrs/010-compute-strategy.md) para el análisis completo.

API Gateway + Lambda, sin cómputo permanentemente activo (App Runner/ECS descartados para V1) — la carga es principalmente request-driven y el trabajo largo es asíncrono.

## Estrategia de IaC

Ver [ADR-002](adrs/002-iac-strategy.md) para la justificación completa.

- **Terraform (HCL)** es la herramienta principal de IaC para toda la infraestructura de DocLens: Route 53, CloudFront, WAF, S3, Cognito, API Gateway, Lambda, Aurora Serverless v2, EventBridge, SQS/DLQ, SNS, Textract, Bedrock Knowledge Bases, IAM, cifrado y observabilidad.
- Estado de Terraform: backend S3 + tabla de bloqueo DynamoDB.
- El código de Terraform vive en `infra/terraform/` (proyecto Lambda) y `infra/` (plantilla Web).

## Decisiones Arquitectónicas Clave

| Decisión | Alternativa descartada | Justificación | ADR |
|---|---|---|---|
| Edge nativo de AWS | Cloudflare | Menor complejidad, un solo proveedor | [009](adrs/009-edge-network-strategy.md) |
| API Gateway + Lambda | App Runner / ECS | No se necesita cómputo permanente | [010](adrs/010-compute-strategy.md) |
| Subida vía URL S3 prefirmada | Subida a través del backend | Durabilidad, tamaño y costo | [005](adrs/005-upload-strategy.md) |
| Procesamiento asíncrono | Procesamiento síncrono | Operaciones largas y resiliencia | [006](adrs/006-sync-vs-async-processing.md) / [011](adrs/011-document-processing-trigger.md) |
| OCR híbrido | Textract siempre | Costo y latencia | [003](adrs/003-ocr-strategy.md) |
| Bedrock Knowledge Bases | RAG manual | Menos código y mantenimiento | [001](adrs/001-rag-strategy.md) |
| Aurora + pgvector | Otro vector store separado | Reutiliza la base de datos existente, soportado por Bedrock | [008](adrs/008-data-storage-strategy.md) |
| Sin agente en V1 | Agente autónomo | El alcance actual no lo requiere | [012](adrs/012-no-autonomous-agent.md) |
| Step Functions + idempotencia por nombre de ejecución | SQS+DLQ coordinado a mano entre Lambdas / ejecución directa | Recuperación frente a fallos, sin reimplementar "esperar" tres veces | [011](adrs/011-document-processing-trigger.md) |
| GuardDuty → EventBridge | Sondeo de tag con reintentos | Elimina la ventana de "escaneo aún no listo" | [011](adrs/011-document-processing-trigger.md) |
| Región única (`eu-west-1`) | Despliegue activo multi-región | Complejidad y costo no justificados para V1 | [013](adrs/013-disaster-recovery-strategy.md) |
| Redacción de PII antes de la ingesta al KB | Hashing de campos conocidos | El KB nunca guarda PII en texto plano; hashing es reversible por fuerza bruta en valores de baja entropía | [014](adrs/014-pii-redaction.md) |

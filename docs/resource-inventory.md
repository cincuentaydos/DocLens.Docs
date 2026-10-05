# Inventario de Recursos — `sbx`

!!! note "Foto de un momento, no estado en vivo"
    Este documento es una **instantánea**, actualizada por última vez el **2026-09-29** contra la cuenta AWS dedicada del proyecto (`<ACCOUNT_ID>`, región `eu-west-1`) — el root `infra/terraform` de `DocLens.Backend` se revisó en esa fecha (pipeline ADR-011/014 completo); `DocLens.Infra` (raíz y `governance/`) sigue reflejando el 2026-09-28/2026-09-12 respectivamente. No se regenera automáticamente — la fuente de verdad en todo momento es `terraform state list` / `terraform output` de cada uno de los tres roots de Terraform. Si se aplican cambios después de esta fecha, este inventario queda desactualizado hasta la próxima revisión manual.

## Resumen

| Root de Terraform | Repo | Recursos gestionados |
|---|---|---|
| `main.tf` (raíz) | `DocLens.Infra` | 41 |
| `governance/` | `DocLens.Infra` | 18 |
| `infra/terraform/` | `DocLens.Backend` | 87 |
| **Total** | | **146** (incluye datos auxiliares no persistentes: `aws_caller_identity`, `terraform_remote_state`, `tls_certificate`) |

Cuenta única, sin AWS Organizations — ver [ADR-013](adrs/013-disaster-recovery-strategy.md) y la definición de V1.

## `DocLens.Infra` — stack de plataforma

### Red (`modules/network`)

| Recurso | Identificador |
|---|---|
| VPC | `doclens-sbx` |
| Subnet (AZ 1) | `10.42.0.0/24` |
| Subnet (AZ 2) | `10.42.1.0/24` |
| DB Subnet Group | `doclens-aurora-sbx` |
| Security Group (Aurora) | `doclens-aurora-sbx` — sin reglas de ingreso (RDS Data API, no VPC networking) |

### Identidad (`modules/auth`)

| Recurso | Identificador |
|---|---|
| Cognito User Pool | `eu-west-1_h6nZdU3S7` (`doclens-users-sbx`) |
| User Pool Client | `7gkvn2dtvtlkn5r9kbaeun9snr` |

### Cifrado (`modules/kms`)

| Recurso | Identificador |
|---|---|
| KMS Key | `arn:aws:kms:eu-west-1:<ACCOUNT_ID>:key/58688ef5-a086-48dc-9447-f5ff57e9f14e` |
| KMS Alias | `alias/doclens-app-data-sbx` |

Cifra: almacenamiento de Aurora, el secret del master user, y el bucket de documentos. El bucket de frontend queda en SSE-S3 (AES256) — ver [ADR-008](adrs/008-data-storage-strategy.md) y el header comment de `modules/kms/main.tf`.

### Datos (`modules/data`)

| Recurso | Identificador |
|---|---|
| Aurora Cluster | `doclens-sbx` — PostgreSQL 16.10, Serverless v2 (0.5–4 ACU) |
| Aurora Instance | `doclens-sbx-1` |
| Endpoint | `doclens-sbx.cluster-cdieemoauqel.eu-west-1.rds.amazonaws.com` |
| Secret (master user) | `rds!cluster-06b73bb9-f181-4d50-a533-b21df32aee45-OJR2Va` |

Esquema `kb` (extensión `vector`, tabla `kb.embeddings`, índices HNSW + GIN) creado manualmente vía RDS Data API — no es un recurso de Terraform, ver [ADR-008](adrs/008-data-storage-strategy.md).

### Documentos (`modules/documents`)

| Recurso | Identificador |
|---|---|
| Bucket S3 (originales) | `doclens-documents-<ACCOUNT_ID>-sbx` — versionado, SSE-KMS |
| Bucket S3 (contenido redactado para el KB) | `doclens-kb-source-<ACCOUNT_ID>-sbx` — SSE-KMS, sin versionado, bucket separado a propósito (ver [ADR-014](adrs/014-pii-redaction.md)) |
| GuardDuty Malware Protection Plan | sobre el bucket de documentos (originales únicamente) |
| IAM Role (GuardDuty) | `doclens-guardduty-malware-sbx` |

### IA (`modules/knowledge_base`)

| Recurso | Identificador |
|---|---|
| Bedrock Knowledge Base | `XEAPG1O8J0` (`doclens-documents-sbx` — nombre del recurso, no cambia aunque el data source sí) |
| Data Source | `doclens-kb-source-sbx` (S3, apunta a `doclens-kb-source-<ACCOUNT_ID>-sbx` — nunca al bucket de originales, ver [ADR-014](adrs/014-pii-redaction.md)) |
| IAM Role | `doclens-kb-role-sbx` — solo lectura sobre `doclens-kb-source-<ACCOUNT_ID>-sbx` |

Modelo de embeddings: `amazon.titan-embed-text-v2:0` (1024 dimensiones).

### Edge (`modules/edge`)

| Recurso | Identificador |
|---|---|
| CloudFront Distribution | `https://dwqkm79ishahl.cloudfront.net` |
| Bucket S3 (frontend) | `doclens-frontend-<ACCOUNT_ID>-sbx` — build de `DocLens.Web.Template` desplegado |
| WAF WebACL | `doclens-cloudfront-sbx` (us-east-1) — Common Rule Set + rate limit 2000 req/5min por IP |

**Sin dominio propio todavía** — `enable_custom_domain=false` (ver `envs/sbx.tfvars`): sin certificado ACM ni registro Route 53, la distribución se sirve directamente por su dominio `*.cloudfront.net`. Pendiente de un caso de soporte de AWS por una restricción de registro de dominio en una cuenta nueva.

## `DocLens.Infra/governance` — guardrails de cuenta

| Recurso | Identificador |
|---|---|
| IAM Group | `doclens-developers` |
| Policy attachment | `AdministratorAccess` (AWS managed) |
| Policy inline (guardrail) | `doclens-developers-guardrail` — deniega escalación de privilegios IAM, billing/cierre de cuenta, y todo lo que no sea autogestión de MFA/password sin sesión MFA |
| IAM Users | 3 usuarios developer — identidad + membresía al grupo, sin credenciales gestionadas por Terraform |
| OIDC Provider | `token.actions.githubusercontent.com` — GitHub Actions, sin access keys de larga duración |
| IAM Role (CD web) | `doclens-github-actions-web-deploy-sbx` — `s3:PutObject`/`DeleteObject`/`ListBucket` en el bucket de frontend, `cloudfront:CreateInvalidation` en la distribución |
| IAM Role (CD backend) | `doclens-github-actions-lambda-deploy-sbx` — `lambda:UpdateFunctionCode` en las 3 Lambdas, `s3:PutObject`/`GetObject` en el prefijo `ci/` del bucket de artifacts |

Usuario admin de cuenta (`AdministratorAccess`, fuera del grupo `developers`) creado manualmente, no por Terraform.

## `DocLens.Backend` — backend (`modules/processing`)

| Recurso | Identificador |
|---|---|
| API Gateway (HTTP API) | `mljtp2uzqb` — `https://mljtp2uzqb.execute-api.eu-west-1.amazonaws.com/` |
| JWT Authorizer | Cognito (`doclens-users-sbx`) |
| Lambda — API | `doclens-api-sbx` |
| Bucket S3 (artifacts) | `doclens-lambda-artifacts-<ACCOUNT_ID>-sbx` — staging del ZIP de deploy (self-contained, >50MB) antes de que Terraform lo suba a las funciones |
| CloudWatch Log Groups | uno por función Lambda, retención 30 días |

Swagger UI en `/swagger` — única ruta de API Gateway sin JWT (documentación, no datos); `TenantMiddleware` también la excluye de la resolución de tenant.

### Pipeline ADR-011/014 — 3 Step Functions + 4 Lambdas "worker"

Implementado y validado E2E el 2026-09-29 (ver la entrada correspondiente en la [Bitácora del Despliegue](infra-deployment-log.md)). Un solo ZIP compartido entre las 7 funciones Lambda de este módulo (API + 4 workers + las 2 heredadas de abajo) — el `handler` de Terraform es lo único que cambia por función.

| Recurso | Identificador |
|---|---|
| Step Functions — Extraction | `doclens-document-extraction-sbx` — `DetectFormat → [StartTextractJob] → ClassifyDocument → SemanticExtraction → DetectAndRedactPII` |
| Step Functions — Review | `doclens-document-review-sbx` — `EnterReview` (`.waitForTaskToken`, resuelto por `POST .../review`) `→ Choice(approved)` |
| Step Functions — KB Sync | `doclens-document-kb-sync-sbx` — `PrepareContent → StartIngestionJob → Wait/GetIngestionJob (polling, sin evento nativo de Bedrock) → WriteIndexed/WriteFailed` |
| Lambda — Extraction worker | `doclens-extraction-worker-sbx` — 1024MB/120s, action-routed (`DetectFormat`/`StartTextractJob`/`ClassifyDocument`/`SemanticExtraction`/`DetectAndRedactPII`) |
| Lambda — Textract bridge | `doclens-textract-bridge-sbx` — 256MB/60s, disparado por SNS, resuelve el task token de `StartTextractJob` vía la tabla `textract_jobs` |
| Lambda — Review worker | `doclens-review-worker-sbx` — 256MB/30s |
| Lambda — Sync worker | `doclens-sync-worker-sbx` — 256MB/60s |
| EventBridge Rule | `doclens-guardduty-clean-sbx` — GuardDuty scan limpio → `StartExecution` en Extraction (retargeteado desde SQS) |
| EventBridge Rule | `doclens-document-reprocess-requested-sbx` — evento propio (`POST .../reprocess`) → `StartExecution` en Extraction |
| EventBridge Rule | `doclens-document-extracted-sbx` — evento propio → `StartExecution` en Review |
| EventBridge Rule | `doclens-document-approved-sbx` — evento propio → `StartExecution` en KB Sync |
| SQS — DLQs de EventBridge | `doclens-extraction-trigger-dlq-sbx`, `doclens-review-trigger-dlq-sbx`, `doclens-kb-sync-trigger-dlq-sbx` — solo DLQ, reintento nativo de EventBridge en vez de cola consumidora |
| SNS — Textract completion | `doclens-textract-completion-sbx` |

Tablas Aurora nuevas (RDS Data API, no Terraform — mismo patrón manual que `documentos`/`documento_versiones`): `pii_mappings` (placeholder ↔ valor real, ADR-014), `document_reviews` (task token + auditoría de quién aprobó/rechazó qué), `textract_jobs` (correlación `job_tag` ↔ task token, ya que el `JobTag` de Textract tiene un límite de 64 caracteres, insuficiente para un token de Step Functions).

### Lambdas heredadas (pendientes de limpieza)

| Recurso | Identificador |
|---|---|
| Lambda — Processor | `doclens-processor-sbx` — superseded por `extraction_worker`, sin tráfico desde el retargeteo de `guardduty-clean-sbx` |
| Lambda — OCR result | `doclens-ocr-result-sbx` — superseded por `textract_bridge` |
| SQS — processing | `doclens-processing-sbx` (+ DLQ) — sin consumidor real desde el retargeteo |
| SQS — ocr-result | `doclens-ocr-result-sbx` (+ DLQ) |

Se dejan a propósito en pie por ahora — la limpieza (borrarlos) queda para un `terraform plan`/`apply` propio y separado, nunca mezclado con un cambio que agrega funcionalidad (ver la Bitácora del Despliegue).

## Fuera de alcance en esta instantánea

- **AWS CloudTrail / AWS Config / AWS Budgets** — no implementados; ver el [Plan de Gobernanza](governance-plan.md).
- **Dominio propio** (`doclens.org`) — registro fallido repetidamente vía Route 53 por restricción de cuenta nueva, caso de soporte abierto.

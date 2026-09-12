# Bitácora del Despliegue de Infra — `sbx`

Registro del trabajo de infraestructura hecho entre el **2026-09-09** y el **2026-09-12**: creación de la cuenta AWS del proyecto, división del Terraform en repos, y primer despliegue real de `sbx`. Este documento existe para dejar registro de las decisiones y de los problemas reales encontrados durante un `apply` contra una cuenta real — útil tanto como referencia operativa como para la documentación del trabajo final.

## Contexto

Cuenta AWS dedicada, creada específicamente para este proyecto (no la cuenta personal del autor) — ver la justificación en `architecture.md`. Cuenta única, sin AWS Organizations, región primaria `eu-west-1`.

## Decisiones de arquitectura

### 1. Repo de infra separado, con un límite específico

Se decidió sacar el Terraform que ya existía dentro de `DocLens.Lambda.Template/infra/terraform` a un repo nuevo, `DocLens.Infra`, pero **no todo** — el módulo `processing` (las Lambdas del backend, API Gateway, colas) se quedó en `DocLens.Lambda.Template`.

Razón: `network`, `auth`, `data`, `documents`, `knowledge_base` y `edge` son infraestructura de **plataforma compartida** — no le pertenecen ni al backend ni al frontend específicamente (Cognito, Aurora y el bucket de documentos los usan varios componentes; `edge` sirve tanto el frontend como el backend). `processing` es distinto: es literalmente el despliegue del código del backend, con su propio ciclo de vida de release.

Esto genera una dependencia cruzada de un solo sentido entre los dos states de Terraform, resuelta con `terraform_remote_state` (ver `DocLens.Infra/README.md` → "Cross-repo wiring"):

- `DocLens.Lambda.Template` lee de `DocLens.Infra` (bucket de documentos, Aurora, Cognito, la KMS key).
- `DocLens.Infra` lee de `DocLens.Lambda.Template` (el `api_endpoint`, que `edge` necesita como origin de CloudFront).

El orden de primer despliegue queda documentado en ambos READMEs: `bootstrap` → `DocLens.Infra` sin `edge` → `DocLens.Lambda.Template` → `DocLens.Infra` otra vez (para completar `edge`).

La migración de `DocLens.Lambda.Template/infra/terraform` hacia `DocLens.Infra` se hizo con `git subtree split`, preservando el historial de commits de los archivos movidos.

### 2. Cifrado con una sola CMK compartida

Se agregó un módulo `modules/kms` con **una** key para Aurora + el bucket de documentos, en vez de una key por recurso — no había ningún requisito de aislamiento que justificara múltiples keys, y cada CMK adicional es otro costo mensual y otra policy que mantener. El bucket de frontend se deja en SSE-S3 (AES256): solo tiene assets estáticos, no datos de casos/clientes.

### 3. Gobernanza como Terraform root separado

`governance/` (dentro de `DocLens.Infra`) tiene su propio state — cambia con otra cadencia que el stack de la app y no debería acoplarse a un deploy. Contiene:

- Grupo `doclens-developers` con `AdministratorAccess` + dos guardrails de Deny explícito (siempre ganan sobre cualquier Allow): no pueden escalar privilegios IAM ni tocar billing/cierre de cuenta.
- Un tercer Deny bloquea **todo** excepto autogestión de MFA/password si la sesión no tiene MFA activo (patrón documentado de AWS).
- Los usuarios IAM se crean sin login profile ni access key — Terraform no genera secretos que terminen en el state; las credenciales las emite un admin manualmente por consola, por persona.

## Cronología del primer despliegue real

Cada uno de estos problemas se descubrió recién al correr `terraform apply` contra la cuenta real — quedan documentados porque el mensaje de error de Terraform/AWS no siempre es obvio.

| # | Problema | Causa | Fix |
|---|---|---|---|
| 1 | Push a git rechazado (~1.4 GB) | `DocLens.Infra` nunca tuvo `.gitignore` — `terraform init` local dejó el binario del provider AWS trackeado en un commit | `.gitignore` + `git rm --cached` + `commit --amend` |
| 2 | `CreateSecurityGroup` — `InvalidParameterValue` | La descripción del security group de Aurora tenía un guión largo "—" (no ASCII); EC2 exige ASCII en `GroupDescription` | Reemplazado por un guión simple "-" |
| 3 | `CreateMalwareProtectionPlan` — 400 | Al rol de GuardDuty le faltaban `s3:GetBucketNotification`/`s3:PutBucketNotification` (necesarios para configurar la notificación EventBridge del bucket) | Agregado el statement IAM |
| 4 | `CreateDBCluster` — `Cannot find version 16.6` | La versión de motor Aurora PostgreSQL fijada (`16.6`) no existe en el catálogo actual de RDS | Cambiada a `16.10` (verificado con `aws rds describe-db-engine-versions`) |
| 5 | `CreateMalwareProtectionPlan` — 400 (de nuevo) | Al mismo rol le faltaban además `events:PutRule`/`events:PutTargets` (GuardDuty gestiona su propia regla de EventBridge) | Agregado el statement IAM |
| 6 | `CreateKnowledgeBase` — `rds:DescribeDBClusters` denegado | El rol del Knowledge Base solo tenía permisos `rds-data:*` (Data API), no `rds:DescribeDBClusters` (Bedrock valida la config de storage describiendo el cluster) | Agregado el statement IAM |
| 7 | `CreateKnowledgeBase` — `relation "kb.embeddings" does not exist` | La migración de esquema (`CREATE EXTENSION vector`, esquema `kb`, tabla `kb.embeddings`) es manual por diseño (ADR-008) — nunca se había corrido | Ejecutada vía RDS Data API (`aws rds-data execute-statement`) |
| 8 | `CreateKnowledgeBase` — `chunk_texto column must be indexed` | Bedrock exige un índice GIN de full-text sobre la columna de texto, además del índice HNSW sobre el vector | Índice agregado (`CREATE INDEX ... USING gin (to_tsvector(...))`) |
| 9 | Registro de `doclens.org` en Route 53 — `FAILED` (3 intentos, con root y con IAM admin) | Restricción de antifraude de AWS en cuentas recién creadas — no depende del usuario IAM que lo pide | Caso de soporte de AWS abierto; mientras tanto, `edge` se hizo dominio-opcional (`enable_custom_domain`) para no bloquear el resto del despliegue |

## Estado al cierre de esta bitácora (2026-09-12)

Desplegado y funcionando en `sbx`: red, Cognito, Aurora + pgvector, KMS, bucket de documentos + GuardDuty, Knowledge Base de Bedrock, backend completo (API + Processor + OCR Lambdas, API Gateway, colas), CloudFront + WAF + bucket de frontend (sin dominio propio todavía), y los 3 usuarios developers con sus guardrails. Ver [Inventario de Recursos](resource-inventory.md) para el detalle completo.

Pendiente: desplegar el build de `DocLens.Web.Template` al bucket de frontend, ajustar el CORS del bucket de documentos al dominio real una vez resuelto el registro, budget con alarma, CloudTrail/Config.

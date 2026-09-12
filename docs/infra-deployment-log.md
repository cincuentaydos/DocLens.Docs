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

### 4. CI/CD de código vía GitHub Actions + OIDC, separado del apply de infra

Tanto `DocLens.Web.Template` como `DocLens.Lambda.Template` tienen su propio workflow `CD` (push a la rama principal, o disparo manual eligiendo ambiente). Ninguno de los dos corre `terraform apply` — solo despliegan código:

- **Web**: build → `aws s3 sync` al bucket de frontend → invalidación de CloudFront.
- **Lambda**: `dotnet test` → `dotnet publish` self-contained → sube el ZIP a un bucket S3 de artifacts (necesario porque un build self-contained de .NET supera los 50MB del límite de subida directa de `UpdateFunctionCode`) → `aws lambda update-function-code` en las 3 funciones.

Autenticación contra AWS vía **OIDC** (`aws_iam_openid_connect_provider` + un rol IAM por repo/ambiente en `governance/`), sin access keys de larga duración guardadas como secret de GitHub — el patrón que ya pedía `architecture.md`. Cambios de infraestructura (rutas, roles, colas) siguen yendo por un `terraform apply` revisado aparte.

## Estado al cierre de esta bitácora (2026-09-12)

Desplegado y funcionando en `sbx`: red, Cognito, Aurora + pgvector, KMS, bucket de documentos + GuardDuty, Knowledge Base de Bedrock, backend completo (API + Processor + OCR Lambdas, API Gateway, colas, Swagger UI en `/swagger`), CloudFront + WAF + bucket de frontend con el build de `DocLens.Web.Template` ya desplegado (sin dominio propio todavía), los 3 usuarios developers con sus guardrails, y pipelines de CI/CD funcionando de punta a punta en ambos repos de aplicación. Ver [Inventario de Recursos](resource-inventory.md) para el detalle completo.

Pendiente: dominio propio (`doclens.org` — registro vía Route 53 falló repetidamente por una restricción de cuenta nueva; caso de soporte de AWS abierto), budget con alarma, CloudTrail/Config.

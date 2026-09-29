# Bitácora del Despliegue de Infra — `sbx`

Registro del trabajo de infraestructura hecho entre el **2026-09-09** y el **2026-09-12**: creación de la cuenta AWS del proyecto, división del Terraform en repos, y primer despliegue real de `sbx`. Este documento existe para dejar registro de las decisiones y de los problemas reales encontrados durante un `apply` contra una cuenta real — útil tanto como referencia operativa como para la documentación del trabajo final.

## Contexto

Cuenta AWS dedicada, creada específicamente para este proyecto (no la cuenta personal del autor) — ver la justificación en `architecture.md`. Cuenta única, sin AWS Organizations, región primaria `eu-west-1`.

## Decisiones de arquitectura

### 1. Repo de infra separado, con un límite específico

Se decidió sacar el Terraform que ya existía dentro de `DocLens.Backend/infra/terraform` a un repo nuevo, `DocLens.Infra`, pero **no todo** — el módulo `processing` (las Lambdas del backend, API Gateway, colas) se quedó en `DocLens.Backend`.

Razón: `network`, `auth`, `data`, `documents`, `knowledge_base` y `edge` son infraestructura de **plataforma compartida** — no le pertenecen ni al backend ni al frontend específicamente (Cognito, Aurora y el bucket de documentos los usan varios componentes; `edge` sirve tanto el frontend como el backend). `processing` es distinto: es literalmente el despliegue del código del backend, con su propio ciclo de vida de release.

Esto genera una dependencia cruzada de un solo sentido entre los dos states de Terraform, resuelta con `terraform_remote_state` (ver `DocLens.Infra/README.md` → "Cross-repo wiring"):

- `DocLens.Backend` lee de `DocLens.Infra` (bucket de documentos, Aurora, Cognito, la KMS key).
- `DocLens.Infra` lee de `DocLens.Backend` (el `api_endpoint`, que `edge` necesita como origin de CloudFront).

El orden de primer despliegue queda documentado en ambos READMEs: `bootstrap` → `DocLens.Infra` sin `edge` → `DocLens.Backend` → `DocLens.Infra` otra vez (para completar `edge`).

La migración de `DocLens.Backend/infra/terraform` hacia `DocLens.Infra` se hizo con `git subtree split`, preservando el historial de commits de los archivos movidos.

### 2. Cifrado con una sola CMK compartida

Se agregó un módulo `modules/kms` con **una** key para Aurora + el bucket de documentos, en vez de una key por recurso — no había ningún requisito de aislamiento que justificara múltiples keys, y cada CMK adicional es otro costo mensual y otra policy que mantener. El bucket de frontend se deja en SSE-S3 (AES256): solo tiene assets estáticos, no datos de casos/clientes.

### 3. Gobernanza como Terraform root separado

`governance/` (dentro de `DocLens.Infra`) tiene su propio state — cambia con otra cadencia que el stack de la app y no debería acoplarse a un deploy. Contiene:

- Grupo `doclens-developers` con `AdministratorAccess` + dos guardrails de Deny explícito (siempre ganan sobre cualquier Allow): no pueden escalar privilegios IAM ni tocar billing/cierre de cuenta.
- Un tercer Deny bloquea **todo** excepto autogestión de MFA/password si la sesión no tiene MFA activo (patrón documentado de AWS).
- Los usuarios IAM se crean sin login profile ni access key — Terraform no genera secretos que terminen en el state; las credenciales las emite un admin manualmente por consola, por persona.

### 4. CI/CD de código vía GitHub Actions + OIDC, separado del apply de infra

Tanto `DocLens.Web.Template` como `DocLens.Backend` tienen su propio workflow `CD` (push a la rama principal, o disparo manual eligiendo ambiente). Ninguno de los dos corre `terraform apply` — solo despliegan código:

- **Web**: build → `aws s3 sync` al bucket de frontend → invalidación de CloudFront.
- **Lambda**: `dotnet test` → `dotnet publish` self-contained → sube el ZIP a un bucket S3 de artifacts (necesario porque un build self-contained de .NET supera los 50MB del límite de subida directa de `UpdateFunctionCode`) → `aws lambda update-function-code` en las 3 funciones.

Autenticación contra AWS vía **OIDC** (`aws_iam_openid_connect_provider` + un rol IAM por repo/ambiente en `governance/`), sin access keys de larga duración guardadas como secret de GitHub — el patrón que ya pedía `architecture.md`. Cambios de infraestructura (rutas, roles, colas) siguen yendo por un `terraform apply` revisado aparte.

## Estado al cierre de esta bitácora (2026-09-12)

Desplegado y funcionando en `sbx`: red, Cognito, Aurora + pgvector, KMS, bucket de documentos + GuardDuty, Knowledge Base de Bedrock, backend completo (API + Processor + OCR Lambdas, API Gateway, colas, Swagger UI en `/swagger`), CloudFront + WAF + bucket de frontend con el build de `DocLens.Web.Template` ya desplegado (sin dominio propio todavía), los 3 usuarios developers con sus guardrails, y pipelines de CI/CD funcionando de punta a punta en ambos repos de aplicación. Ver [Inventario de Recursos](resource-inventory.md) para el detalle completo.

Pendiente: dominio propio (`doclens.org` — registro vía Route 53 falló repetidamente por una restricción de cuenta nueva; caso de soporte de AWS abierto), budget con alarma, CloudTrail/Config.

## Entrada — 2026-09-28: bucket separado para el Knowledge Base (ADR-014)

Sesión de arquitectura previa (2026-09-21 a 2026-09-27) rediseñó el pipeline de procesamiento de documentos alrededor de tres AWS Step Functions y de un paso nuevo de redacción de PII antes de la ingesta al Knowledge Base — ver [ADR-011](adrs/011-document-processing-trigger.md) (revisada) y [ADR-014](adrs/014-pii-redaction.md) (nueva). Esta entrada cubre el primer `apply` de esa revisión, aplicado solo contra `main.tf` (raíz) de `DocLens.Infra` — las Step Functions en sí todavía no están implementadas, quedan para una sesión aparte.

**Cambio:** el data source del Bedrock Knowledge Base apuntaba a `doclens-documents-{env}` — el bucket de los documentos originales, con PII en claro. Se creó `doclens-kb-source-{env}` como bucket separado (mismo CMK, sin GuardDuty — nada se sube ahí directamente, sin versionado — es contenido derivado, no fuente de verdad) y se reapuntó tanto el rol IAM del Knowledge Base como su data source hacia ese bucket nuevo. `doclens-documents-{env}` ya no tiene ningún rol con acceso de lectura desde el lado de IA — la garantía "el KB nunca ve PII" pasa a ser estructural (IAM), no una convención de qué prefijo usa cada Lambda.

`terraform plan` contra `sbx`: 5 recursos creados (el bucket nuevo + sus 3 configuraciones + el data source recreado), 1 modificado (la policy IAM del rol del KB), 1 destruido (el data source anterior — Bedrock no permite renombrar un data source in-place, solo recrearlo). Sin contenido indexado todavía en ese momento, así que la recreación no tuvo costo de reingesta. `apply` limpio, sin sorpresas.

**Estado al cierre de esta entrada:** `doclens-kb-source-311844029093-sbx` desplegado y siendo la única fuente que el Knowledge Base puede leer. El resto del pipeline (las 3 Step Functions, la tabla `pii_mappings`, los handlers Lambda de extracción/revisión/sync) sigue sin implementar — esta entrada queda desactualizada en eso hasta la próxima sesión de implementación. Ver [Inventario de Recursos](resource-inventory.md) para el detalle completo y actualizado de lo desplegado.

## Entrada — 2026-09-29: pipeline ADR-011/014 implementado y validado E2E

Implementación completa de lo que quedó pendiente en la entrada anterior: las 3 Step Functions (Extraction, Review, KB Sync), las 4 Lambdas "worker" action-routed, el redireccionamiento de EventBridge, y todo el código C# detrás (clasificación con Bedrock, redacción de PII con Comprehend, servicio de chat con reversión de PII). Detalle completo del diseño en [ADR-011](adrs/011-document-processing-trigger.md) (revisada) y [ADR-014](adrs/014-pii-redaction.md); inventario actualizado en [Inventario de Recursos](resource-inventory.md).

**Primer `apply` completo contra `sbx`:** 50 recursos creados / 4 modificados / 3 destruidos (las Step Functions, las 4 Lambdas nuevas, las reglas de EventBridge, las colas DLQ, y los roles IAM correspondientes).

### Depuración en vivo — 7 bugs reales, en orden

Cada uno solo salió a la luz una vez resuelto el anterior — normal para la primera corrida real de un pipeline nuevo de varias etapas, no señal de un diseño fràgil:

1. **GuardDuty Malware Protection Plan nunca escaneó nada** — su Malware Protection Plan quedó atascado en estado `WARNING`/`INSUFFICIENT_TEST_OBJECT_PERMISSIONS` desde el `apply` original porque su propio rol IAM no tenía `s3:PutObject`/`s3:DeleteObject` ni permisos de KMS para escribir su objeto de auto-validación. Sin esto, GuardDuty jamás pasa a `ACTIVE` y ninguna ejecución de Step Functions llega a dispararse — cero ejecuciones, sin ningún error visible en ningún lado. Fix: agregar esos permisos al rol de GuardDuty en `DocLens.Infra/modules/documents/main.tf`.
2. **Cada Lambda worker fallaba con `Runtime.ExitError` en el arranque** — el bootstrap del custom runtime (`Program.cs`) nunca registraba `IConfiguration` como singleton en el `ServiceCollection` armado a mano, así que cualquier servicio que la pidiera por constructor (`TextractOcrService`, etc.) fallaba al resolver dependencias.
3. **`extraction_worker` sin `kms:GenerateDataKey`** para escribir en el bucket `kb_source` (CMK compartido con `documents`, pero el rol solo tenía `kms:Decrypt`).
4. **El bug más interesante: cada worker "tenía éxito" pero devolvía un cuerpo de respuesta vacío**, rompiendo cualquier `Choice` state que leyera su output (`"Invalid path '...': the choice state's condition path references an invalid value"`). Causa raíz: `LambdaBootstrapBuilder.Create<TInput>(...)` con un solo type-argument explícito compila igual — pero, por covarianza de delegados en C#, se resuelve contra una sobrecarga distinta que descarta el valor de retorno del handler en vez de serializarlo. Se reprodujo el bug de forma aislada (una Runtime API de Lambda falsa corriendo localmente) antes de tocar el despliegue real, para confirmar la causa antes de aplicar el fix. Solución: especificar ambos type-arguments explícitamente, `Create<TInput, TOutput>(...)`.
5. **`extraction_worker` sin `s3:GetObject` en `kb_source`** — tenía permiso de escritura (`PutObject`) para el texto intermedio, pero no de lectura para releerlo en el paso siguiente (`ClassifyDocument`).
6. **Bedrock/Claude envolvía su respuesta JSON en una cerca de código markdown** (` ```json ... ``` `) pese a que el prompt pedía explícitamente no hacerlo — rompía el segundo parseo JSON en `BedrockSemanticAnalysisService`. Solución: función auxiliar que despoja la cerca antes de deserializar.
7. **La Lambda API (que sirve `/chat`) nunca recibió `KNOWLEDGE_BASE_ID` en su entorno** — solo se le había pasado a `sync_worker`.

### Resultado

Primer flujo end-to-end exitoso confirmado el mismo día: subida de documento → escaneo GuardDuty → Extraction (`SUCCEEDED`) → Review (`SUCCEEDED`, resuelto vía `POST .../review`) → KB Sync (`SUCCEEDED`) → `POST /chat` devolvió una respuesta con fuente citada (`documentoId` correcto) y con la PII correctamente revertida a su valor real en la respuesta (quedó redactada en el Knowledge Base, como exige ADR-014).

**Pendiente, no bloqueante:** limpieza de `doclens-processor-sbx`/`doclens-ocr-result-sbx` y sus colas (quedan sin tráfico pero desplegados — ver [Inventario de Recursos](resource-inventory.md)); parseo nativo de `.docx`/`.pptx`/`.xlsx` (ADR-003, sigue lanzando `NotSupportedException`); aislamiento de chat a nivel de `proceso_id` (necesita las tablas `procesos`/`permisos`, que no existen todavía).

# Plan de Gobernanza — CloudTrail, AWS Config y Budgets

Plan de implementación de los tres guardrails de cuenta que `DocLens.Infra/governance/README.md` lista como "Not yet implemented". Todo va en el root de Terraform `governance/` (state propio, aplicado por el admin — los developers no pueden aplicarlo porque crea roles y policies IAM, que su grupo tiene denegado).

## Contexto y restricciones

- **Cuenta única, sin AWS Organizations** → no hay SCPs. Lo único que protege los recursos de auditoría frente a un developer (que tiene `AdministratorAccess`) son Deny explícitos en la policy del grupo `doclens-developers`. Cada fase de abajo agrega el suyo; sin él, un developer podría apagar el trail o borrar los logs que lo auditan.
- **Dos regiones con recursos**: `eu-west-1` (todo el stack) y `us-east-1` (certificado ACM y WAF de CloudFront, ADR-009). Lo que sea regional (Config) tiene que cubrir las dos.
- **RGPD**: el objetivo principal es poder responder *quién hizo qué y cuándo* sobre la cuenta y sobre los documentos de clientes, y detectar desvíos de configuración (bucket público, cifrado desactivado).

## Orden propuesto

| Fase | Qué | Por qué en este orden | Esfuerzo |
|---|---|---|---|
| 1 | Budgets + Cost Anomaly Detection | Lo más barato y rápido; hoy no hay ninguna alarma de costo | ~½ día |
| 2 | CloudTrail + bucket de auditoría | Base de la auditoría; Config reutiliza el mismo bucket | ~1 día |
| 3 | AWS Config + reglas gestionadas | Depende del bucket de la fase 2 | ~1–2 días |

Un PR por fase en `DocLens.Infra`, cada uno con su `terraform plan` revisado antes del `apply`.

## Fase 1 — AWS Budgets

**Recursos** (`governance/budgets.tf`):

- `aws_budgets_budget` **mensual de costo total de la cuenta**, con tres notificaciones por email:
    - 50 % del real (aviso temprano),
    - 80 % del real,
    - 100 % del **pronosticado** (avisa antes de pasarse, no después).
- `aws_budgets_budget` **por ambiente** (`sbx`, `prod`), filtrado por el tag `Environment` que ya ponen los `default_tags` del stack de la app. Requiere activar el tag como *cost allocation tag* (`aws_ce_cost_allocation_tag`, `status = "Active"`); los costos solo se etiquetan desde la activación en adelante, no retroactivamente.
- `aws_ce_anomaly_monitor` (tipo `DIMENSIONAL`, por servicio) + `aws_ce_anomaly_subscription` con umbral absoluto (p. ej. 20 USD) → email. Detecta picos (Bedrock, Textract) que un budget mensual vería tarde.

**Variables nuevas**: `budget_monthly_limit_usd`, `budget_env_limits_usd` (map), `alert_emails` (list).

**Guardrail**: ya existe — `DenyBillingAndAccount` deniega `budgets:ModifyBudget` y `budgets:DeleteBudget`. Agregar `ce:DeleteAnomalyMonitor`, `ce:UpdateAnomalyMonitor`, `ce:DeleteAnomalySubscription`, `ce:UpdateAnomalySubscription`.

**Costo**: budgets sin *actions* y Cost Anomaly Detection no tienen costo.

**Validación**: el email de confirmación de la suscripción llega a cada destinatario; el budget aparece en *Billing → Budgets* con el gasto actual.

## Fase 2 — AWS CloudTrail

**Recursos** (`governance/cloudtrail.tf`):

- **Bucket de auditoría** `doclens-audit-logs-<ACCOUNT_ID>`, compartido con Config (prefijos `cloudtrail/` y `config/`):
    - Block Public Access completo, `BucketOwnerEnforced`, versionado.
    - **Object Lock** habilitado al crear el bucket (no se puede activar después) en modo `GOVERNANCE` con retención por defecto (ver decisiones abiertas). Ni el admin puede borrar un log antes de tiempo sin el permiso explícito `s3:BypassGovernanceRetention`.
    - Bucket policy: escritura solo para `cloudtrail.amazonaws.com` y `config.amazonaws.com` con condición `aws:SourceArn`/`aws:SourceAccount`; Deny de transporte sin TLS (mismo patrón `DenyNonSSL` de `modules/documents`).
    - Lifecycle: a Glacier Instant Retrieval a los 90 días, expiración al final de la retención.
- **CMK dedicada** `alias/doclens-audit` (no la CMK compartida del stack de la app: esa es por ambiente y la administra otro state). Key policy que permite a CloudTrail cifrar y al admin descifrar; los developers solo lectura vía consola de CloudTrail (Event history no necesita la key).
- `aws_cloudtrail` **multi-región**, `include_global_service_events = true`, `enable_log_file_validation = true`, `kms_key_id` = CMK de arriba.
    - **Management events** (lectura y escritura): toda la actividad de control de la cuenta.
    - **Data events** solo en los buckets de documentos y del Knowledge Base (`s3:GetObject`/`PutObject`/`DeleteObject`) — es lo que responde "quién leyó el documento de qué cliente". Activar en `prod` siempre; en `sbx` solo si el costo lo permite (decisión abierta).
- **Alertas de seguridad** (opcional dentro de la fase, recomendado): trail también a CloudWatch Logs (retención 90 días) + metric filters y alarmas → SNS → email para: login de root, login de consola sin MFA, cambios de policies IAM, `StopLogging`/`DeleteTrail`, cambios en la key policy de la CMK de auditoría.

**Guardrail nuevo** en `doclens-developers` (`DenyAuditTampering`):

- `cloudtrail:StopLogging`, `DeleteTrail`, `UpdateTrail`, `PutEventSelectors`, `PutInsightSelectors`;
- en el bucket de auditoría: `s3:DeleteBucket`, `DeleteBucketPolicy`, `PutBucketPolicy`, `PutLifecycleConfiguration`, `PutBucketObjectLockConfiguration`, `PutBucketVersioning`, `DeleteObject*`, `BypassGovernanceRetention`;
- en la CMK de auditoría: `kms:ScheduleKeyDeletion`, `DisableKey`, `PutKeyPolicy`, `CreateGrant`, `Decrypt`;
- en el log group: `logs:DeleteLogGroup`, `PutRetentionPolicy`, `DeleteMetricFilter`.

**Costo**: la primera copia de management events es gratis; data events 0,10 USD por cada 100 000 eventos; CMK ~1 USD/mes; S3 marginal. Con el volumen actual, unos pocos USD/mes.

**Validación**: `aws cloudtrail get-trail-status` → `IsLogging: true`; `aws cloudtrail validate-logs` sobre las últimas horas; descargar un documento en `sbx` y ver el `GetObject` en el log; como developer, `aws cloudtrail stop-logging` debe dar `AccessDenied`.

## Fase 3 — AWS Config

**Recursos** (`governance/config.tf`):

- `aws_iam_service_linked_role` para `config.amazonaws.com` (`AWSServiceRoleForConfig`).
- En **cada región** (`eu-west-1` y `us-east-1`, con un provider alias como ya hace el stack de la app):
    - `aws_config_configuration_recorder` con todos los tipos soportados; **recursos globales (IAM) solo en `eu-west-1`** para no duplicar *configuration items*.
    - Frecuencia de registro **diaria** (`recording_mode`) en `sbx` para bajar costo; continua en `prod`.
    - `aws_config_delivery_channel` → bucket de auditoría, prefijo `config/`, snapshot cada 24 h.
    - `aws_config_configuration_recorder_status` habilitado.
- **Reglas gestionadas** (`aws_config_config_rule`), elegidas por los requisitos RGPD/auditoría:

| Área | Reglas |
|---|---|
| S3 | `s3-bucket-level-public-access-prohibited`, `s3-bucket-public-read-prohibited`, `s3-bucket-public-write-prohibited`, `s3-bucket-server-side-encryption-enabled`, `s3-bucket-ssl-requests-only`, `s3-bucket-versioning-enabled` |
| Cifrado / datos | `rds-storage-encrypted`, `rds-cluster-deletion-protection-enabled`, `kms-cmk-not-scheduled-for-deletion`, `sqs-queue-encrypted` |
| Identidad | `root-account-mfa-enabled`, `iam-root-access-key-check`, `mfa-enabled-for-iam-console-access`, `access-keys-rotated` (90 días), `iam-user-unused-credentials-check` |
| Auditoría | `multi-region-cloudtrail-enabled`, `cloud-trail-log-file-validation-enabled`, `cloud-trail-encryption-enabled` |
| Cómputo / borde | `lambda-function-public-access-prohibited`, `api-gw-associated-with-waf` (si aplica), `cloudfront-viewer-policy-https` (solo `us-east-1`) |

- **Notificación**: regla de EventBridge sobre `Config Rules Compliance Change` con `newEvaluationResult.complianceType = NON_COMPLIANT` → el mismo SNS de la fase 2. Sin remediación automática en V1: el hallazgo se revisa y se corrige vía Terraform.

**Guardrail nuevo**: `config:StopConfigurationRecorder`, `DeleteConfigurationRecorder`, `PutConfigurationRecorder`, `DeleteDeliveryChannel`, `PutDeliveryChannel`, `DeleteConfigRule`, `PutConfigRule`, `DeleteEvaluationResults`, más `events:DeleteRule`/`DisableRule`/`RemoveTargets` sobre la regla de notificación.

**Costo**: 0,003 USD por *configuration item* registrado + 0,001 USD por evaluación de regla. Con un stack serverless chico y registro diario en `sbx`, del orden de 5–15 USD/mes en total; revisar en *Cost Explorer* la primera semana y, si hace falta, excluir tipos de recurso muy ruidosos.

**Validación**: dashboard de Config muestra recursos de ambas regiones; ninguna regla queda en estado `Not evaluated` tras 24 h; los `NON_COMPLIANT` iniciales se revisan uno por uno (corregir o documentar como excepción aceptada).

## Decisiones abiertas

| Decisión | Opciones | Recomendación |
|---|---|---|
| Retención de logs de auditoría | 1 año / 5 años / otra | Confirmar con el requisito legal del despacho; 1 año como mínimo técnico |
| Modo de Object Lock | `GOVERNANCE` / `COMPLIANCE` | `GOVERNANCE` en V1 (`COMPLIANCE` no se puede revertir ni por el root) |
| Data events en `sbx` | Sí / No | No, salvo que `sbx` tenga datos reales de clientes |
| Monto de los budgets | — | Basarse en el gasto real del último mes en Cost Explorer + margen |
| Destinatarios de alertas | — | Lista de emails del equipo; idealmente un alias compartido |

## Residuales fuera de este plan

- `sts:AssumeRole` sobre roles que confíen en la raíz de la cuenta: hoy no hay ninguno en Terraform; Config/CloudTrail permiten detectar si aparece uno.
- Roles de GitHub Actions que aceptan cualquier rama/PR (`repo:...:*`): restringir a la rama de deploy cuando se defina la estrategia de ramas (comentario ya presente en `governance/main.tf`).
- GuardDuty a nivel de cuenta (hoy solo Malware Protection para S3) y Security Hub: candidatos para una fase posterior.

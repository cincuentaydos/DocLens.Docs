# Comparación de IaC: AWS CDK (C#) vs. Terraform

> La decisión formal está en [ADR-002](002-iac-strategy.md). **Se eligió Terraform.**
> Este documento conserva las observaciones reales hechas al construir ambas implementaciones.

---

## Contexto

Ambas implementaciones aprovisionan la misma infraestructura de DocLens:
bucket S3, tabla DynamoDB (exploración inicial), User Pool de Cognito, función Lambda, API Gateway HTTP v2, roles IAM, y grupos de logs de CloudWatch.

La versión CDK vive en `infra/src/DocLens.Infra/`.
La versión Terraform vive en `infra/terraform/`.

---

## Líneas de código

| Capa | CDK (C#) | Terraform (HCL) |
| --- | --- | --- |
| Almacenamiento (S3 + DynamoDB) | 39 líneas | 85 líneas |
| Auth (Cognito) | 55 líneas | 68 líneas |
| Procesamiento (Lambda + API GW + IAM) | 125 líneas | 165 líneas |
| Punto de entrada + stack | 35 líneas | 55 líneas |
| **Total** | **~254 líneas** | **~373 líneas** |

CDK es ~30% más conciso. La diferencia proviene casi enteramente de IAM — CDK genera políticas automáticamente vía `grantRead`, `grantReadWriteData`, y `AddToRolePolicy`. Terraform requiere escribir cada statement de IAM a mano.

---

## Ejemplos lado a lado

### Bucket S3 con cifrado y refuerzo de SSL

=== "CDK (C#)"

    ```csharp
    DocumentBucket = new Bucket(this, "DocumentBucket", new BucketProps
    {
        Encryption = BucketEncryption.S3_MANAGED,
        BlockPublicAccess = BlockPublicAccess.BLOCK_ALL,
        EnforceSSL = true,
        RemovalPolicy = RemovalPolicy.RETAIN
    });
    ```

=== "Terraform (HCL)"

    ```hcl
    resource "aws_s3_bucket" "documents" { ... }

    resource "aws_s3_bucket_server_side_encryption_configuration" "documents" { ... }

    resource "aws_s3_bucket_public_access_block" "documents" {
      block_public_acls       = true
      block_public_policy     = true
      ignore_public_acls      = true
      restrict_public_buckets = true
    }

    resource "aws_s3_bucket_policy" "enforce_ssl" {
      policy = jsonencode({
        Statement = [{ Condition = { Bool = { "aws:SecureTransport" = "false" } } ... }]
      })
    }
    ```

`EnforceSSL = true` en CDK genera automáticamente la política del bucket. En Terraform requiere un recurso `aws_s3_bucket_policy` separado con un documento IAM JSON escrito a mano. Esto es representativo de cómo se acumula la diferencia de verbosidad.

---

### Permisos IAM para Lambda

=== "CDK (C#)"

    ```csharp
    props.DocumentBucket.GrantRead(ProcessorFunction);
    props.JobsTable.GrantReadWriteData(ProcessorFunction);
    ```

=== "Terraform (HCL)"

    ```hcl
    resource "aws_iam_role_policy" "lambda_permissions" {
      policy = jsonencode({
        Statement = [
          { Sid = "S3Read", Action = ["s3:GetObject", "s3:GetObjectVersion"], ... },
          { Sid = "DynamoDB", Action = ["dynamodb:GetItem", "dynamodb:PutItem",
            "dynamodb:UpdateItem", "dynamodb:DeleteItem", "dynamodb:Query",
            "dynamodb:Scan"], ... },
          { Sid = "Logs", Action = ["logs:CreateLogStream", "logs:PutLogEvents"], ... },
          { Sid = "XRay", Action = ["xray:PutTraceSegments", ...], ... }
        ]
      })
    }
    ```

CDK infiere el mínimo privilegio automáticamente a partir de los métodos de los constructos L2. Terraform requiere conocer las acciones IAM exactas y escribirlas explícitamente — lo cual es más transparente pero significativamente más verboso, y más fácil de hacer mal.

---

## Problema real encontrado: runtime .NET 10

Al construir la versión Terraform, se encontró el siguiente error de validación:

```
expected runtime to be one of [..., "dotnet8", ...], got "dotnet10"
```

El provider `hashicorp/aws` 5.x aún no incluye `dotnet10` como un valor válido del enum. La solución fue usar `provided.al2023` (runtime personalizado) con un comentario explicando el workaround.

La versión CDK usó `Runtime.DOTNET_8` sin problema — y podría referenciar `dotnet10` como string si fuera necesario, porque CDK pasa el valor directamente a CloudFormation sin validación de enum a nivel de framework.

Este es un ejemplo concreto del rezago del provider descrito en [ADR-002](002-iac-strategy.md). Requirió un workaround el primer día de escribir el Terraform.

---

## Gestión de estado

| Aspecto | CDK | Terraform |
| --- | --- | --- |
| Backend de estado | CloudFormation (gestionado por AWS) | Archivo `.tfstate` (S3 + bloqueo DynamoDB para equipos) |
| Equivalente a `plan` | `cdk diff` | `terraform plan` |
| Visibilidad | Eventos de stack en la consola de CloudFormation | Diff en texto plano en la terminal |
| Detección de drift | Detección de drift de CloudFormation (disparo manual) | `terraform plan` siempre muestra el drift |

La salida de `terraform plan` es más fácil de leer y más accionable que los eventos de stack de CloudFormation. Muestra exactamente qué recursos se crearán, modificarán o destruirán antes de aplicar cualquier cambio.

La detección de drift de CloudFormation debe dispararse manualmente y puede ser lenta. `terraform plan` siempre refleja el estado actual.

---

## Empaquetado de Lambda

| Aspecto | CDK | Terraform |
| --- | --- | --- |
| Build + zip | Automático vía `BundlingOptions` | Manual — debe compilarse y comprimirse antes de `terraform apply` |
| Detección de cambios en el código fuente | CDK rastrea el hash del asset | `filebase64sha256(var.lambda_zip_path)` |
| Integración con CI | Un solo paso `cdk deploy` | Requiere un paso de build antes de `terraform apply` |

La integración de bundling de CDK es una ventaja significativa para proyectos con muchas Lambdas. `Code.FromAsset` con `BundlingOptions` ejecuta `dotnet publish` dentro de un contenedor Docker durante `cdk deploy`, de modo que el pipeline de despliegue es un solo comando.

Con Terraform, un script de build (`dotnet publish` + `zip`) debe ejecutarse antes de `terraform apply`. Esto añade un paso que debe cablearse manualmente en el CI.

---

## Resumen

| Dimensión | CDK (C#) | Terraform |
| --- | --- | --- |
| Líneas de código | ~254 | ~373 |
| Lenguaje | C# (igual que el código Lambda) | HCL (lenguaje separado) |
| Verbosidad de IAM | Baja (grants autogenerados) | Alta (cada acción explícita) |
| Empaquetado de Lambda | Automático | Requiere paso de build manual |
| Soporte de runtime .NET 10 | Sí | No — requiere workaround `provided.al2023` |
| Visibilidad de estado | Eventos de CloudFormation | Diff explícito de `terraform plan` |
| Detección de drift | Disparo manual | Integrada en `terraform plan` |
| Familiaridad del equipo | Baja | Alta |
| Multi-cloud / recursos externos | No | Sí |

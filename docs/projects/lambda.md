# DocLens.Lambda

Núcleo del backend. Compuesto por dos funciones Lambda: una **Lambda de API** que maneja las solicitudes de API Gateway (Procesos, Clientes, Documentos, Usuarios, Permisos, IA), y una **Lambda Processor** que consume eventos de SQS (disparados por GuardDuty → EventBridge, ver [ADR-011](../adrs/011-document-processing-trigger.md)) y ejecuta OCR + extracción semántica.

**Repositorio:** [`DocLens.Lambda.Template`](https://github.com/cincuentaydos/DocLens.Lambda.Template)

## Stack

- .NET 10 (ASP.NET Core Minimal APIs)
- Alojamiento en AWS Lambda vía `Amazon.Lambda.AspNetCoreServer.Hosting`
- AWS Lambda Powertools (Logging, Metrics, Tracing)
- AWSSDK: Textract, BedrockRuntime
- Acceso a datos: Aurora PostgreSQL — ORM/librería de acceso aún no definida (ver [ADR-008](../adrs/008-data-storage-strategy.md#preguntas-abiertas))

## Estructura de Directorios

```
src/
  DocLens.Lambda/
    Context/           # Resolución de tenant (TenantContext, TenantMiddleware)
    Endpoints/         # Definiciones de rutas — Procesos, Clientes, Documentos, Usuarios, Permisos, IA
    Extensions/        # Registro de DI (ServiceCollectionExtensions)
    Models/
      Requests/        # DTOs de entrada (records de C#)
      Responses/       # DTOs de salida (records de C#)
    Options/           # Configuración fuertemente tipada (BedrockOptions)
    Services/
      Procesos/        # ProcesoService
      Clientes/        # ClienteService
      Documentos/      # DocumentoService — orquesta extracción
      Usuarios/        # UsuarioService, permisos y auditoría
      Ocr/             # Envoltorio de Textract + parseo nativo: IContentExtractionService
      Semantic/        # Envoltorio de Bedrock: BedrockSemanticAnalysisService
      Ia/              # Orquestación de consultas y borradores (ver ADR-012)
tests/
  DocLens.Lambda.Tests/
    Services/          # Pruebas unitarias que reflejan la estructura de Services/
infra/
  src/DocLens.Infra/   # AWS CDK (C#) — exploración inicial
  terraform/           # Terraform (fuente de verdad, ver ADR-002)
docs/                  # ADRs y documentación técnica
```

## Punto de Entrada

`Program.cs` — host de Minimal APIs con hosting en Lambda, middleware de tenant, y registro de endpoints.

```csharp
builder.Services.AddAWSLambdaHosting(LambdaEventSource.HttpApi);
builder.Services.AddDocLensServices(builder.Configuration);
app.UseMiddleware<TenantMiddleware>();
app.MapProcesoEndpoints();
app.MapClienteEndpoints();
app.MapDocumentoEndpoints();
app.MapUsuarioEndpoints();
app.MapIaEndpoints();
```

## Flujo de Solicitudes

**Lambda de API** (API Gateway):

```
API Gateway → TenantMiddleware → Endpoints (Procesos/Clientes/Documentos/Usuarios/IA)
                                    → Servicio correspondiente
                                        └── Lectura/escritura en Aurora
```

La subida de un documento solo pasa por esta Lambda para `POST /documents/prepare` (genera la URL prefirmada) — el PUT del archivo va directo a S3 y el procesamiento se dispara sin pasar por esta Lambda (ver [ADR-011](../adrs/011-document-processing-trigger.md)).

**Lambda Processor** (consumidor de SQS, disparada tras GuardDuty → EventBridge):

```
SQS → ProcessorHandler
         → calcular SHA-256 (S3:GetObject)
         → DocumentoService.ExtraerAsync
              ├── IContentExtractionService  (texto crudo — enruta por formato, ver ADR-003)
              └── BedrockSemanticAnalysis    (campos estructurados)
         → escritura en Aurora (COMPLETED / DUPLICATE / REJECTED)
```

## Servicios Clave

| Servicio | Interfaz | Propósito |
|---|---|---|
| `ProcesoService` | `IProcesoService` | Crear/actualizar/consultar procesos; valida permisos de acceso |
| `ClienteService` | `IClienteService` | Crear clientes, asociarlos a procesos, provisionar usuarios de cliente |
| `DocumentoService` | `IDocumentoService` | Genera URLs prefirmadas; orquesta la extracción en la Lambda Processor |
| `UsuarioService` | `IUsuarioService` | Crea/gestiona usuarios internos y de cliente; gestiona permisos y auditoría |
| `IContentExtractionService` | `IContentExtractionService` | Enruta la extracción de texto por formato detectado (ver [ADR-003](../adrs/003-ocr-strategy.md)) — solo en la Lambda Processor |
| `BedrockSemanticAnalysisService` | `ISemanticAnalysisService` | Llama a Bedrock/Claude para extraer campos tipados del texto crudo |
| Servicio de orquestación de IA | — | Implementa el flujo explícito de [ADR-012](../adrs/012-no-autonomous-agent.md): autorizar → retrieve en Bedrock KB → generar con Claude → fuentes |

## Regla de Multi-Tenancy

Todo servicio que toque datos **debe** recibir `ITenantContext` vía inyección de constructor e incluir `empresa_id` en todas las claves de almacenamiento, sentencias de log y objetos de respuesta.

| Capa | Clave |
|---|---|
| S3 | `{tenantId}/documents/{documentId}/v{versionNumber}.{ext}` |
| Aurora | Columna `empresa_id` en cada tabla (ver [ADR-008](../adrs/008-data-storage-strategy.md)) |
| Logs | Campo estructurado `empresa_id` en cada entrada |

## Convenciones de Código

- Records de C# para todos los DTOs.
- Interfaz + implementación para cada servicio.
- Servicios: `Scoped`. Clientes del SDK de AWS: `Singleton` (vía `AddAWSService<T>`).
- Nunca instanciar clientes de AWS directamente — siempre usar DI.
- Logging estructurado con `ILogger<T>` — siempre incluir `empresa_id`, identificadores del recurso (`procesoId`/`documentId`), y contexto de la operación.

## Agregar un Nuevo Endpoint

1. Agregar la ruta al archivo correspondiente en `Endpoints/` (o crear uno nuevo para un área de dominio nueva).
2. Registrarlo en el `Map*Endpoints` correspondiente y llamarlo desde `Program.cs`.
3. Crear los records de solicitud/respuesta en `Models/`.
4. Implementar el servicio correspondiente bajo `Services/`.

## Ejecución Local

```bash
dotnet build
dotnet test
dotnet run --project src/DocLens.Lambda
```

## Despliegue

```bash
# Requiere: dotnet tool install -g Amazon.Lambda.Tools
dotnet lambda deploy-function --project-location src/DocLens.Lambda
```

El despliegue basado en Terraform es la vía principal (ver [ADR-002](../adrs/002-iac-strategy.md)); el scaffolding de CDK en `infra/src/DocLens.Infra/` es de la exploración inicial.

## Referencia de Configuración

Toda la configuración vive en `appsettings.json` y puede sobrescribirse vía variables de entorno en Lambda usando el separador `__` (p. ej. `Bedrock__ModelId`).

| Sección | Clave | Predeterminado | Descripción |
|---|---|---|---|
| `Bedrock` | `ModelId` | `anthropic.claude-3-haiku-20240307-v1:0` | ID del modelo de Bedrock usado para extracción semántica |
| `Bedrock` | `MaxTokens` | `1024` | Máximo de tokens en la respuesta de Bedrock |
| `Logging.LogLevel` | `Default` | `Information` | Nivel de log mínimo para el código de la aplicación |

La vinculación fuertemente tipada la maneja `Options/BedrockOptions.cs`, registrada con `ValidateOnStart()` — un valor mal configurado hace que la Lambda falle rápido en el arranque en frío en lugar de en tiempo de solicitud.

## Pruebas

- Pruebas unitarias en `tests/DocLens.Lambda.Tests/`.
- Simular todas las interfaces del SDK de AWS con NSubstitute — nunca hacer llamadas reales a AWS en las pruebas.
- Cada archivo de prueba de servicio refleja la estructura de `Services/`.

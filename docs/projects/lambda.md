# DocLens.Lambda

Core backend. Composed of two Lambda functions: an **intake Lambda** that handles API Gateway requests and enqueues jobs, and a **worker Lambda** that consumes SQS messages and performs OCR + semantic extraction.

**Repository:** [`DocLens.Lambda.Template`](https://github.com/cincuentaydos/DocLens.Lambda.Template)

## Stack

- .NET 10 (ASP.NET Core Minimal APIs)
- AWS Lambda hosting via `Amazon.Lambda.AspNetCoreServer.Hosting`
- AWS Lambda Powertools (Logging, Metrics, Tracing)
- AWSSDK: Textract, BedrockRuntime

## Directory Layout

```
src/
  DocLens.Lambda/
    Context/           # Tenant resolution (TenantContext, TenantMiddleware)
    Endpoints/         # Route definitions — one file per domain area
    Extensions/        # DI registration (ServiceCollectionExtensions)
    Models/
      Requests/        # Inbound DTOs (C# records)
      Responses/       # Outbound DTOs (C# records)
    Options/           # Strongly-typed config bindings (BedrockOptions)
    Services/
      Extraction/      # Orchestrator: DocumentExtractionService
      Ocr/             # Textract wrapper: TextractOcrService
      Semantic/        # Bedrock wrapper: BedrockSemanticAnalysisService
tests/
  DocLens.Lambda.Tests/
    Services/          # Unit tests mirroring the Services/ layout
infra/
  src/DocLens.Infra/   # AWS CDK (C#)
  terraform/           # Terraform scaffolding
docs/                  # ADRs and technical docs
```

## Entry Point

`Program.cs` — Minimal APIs host with Lambda hosting, tenant middleware, and endpoint registration.

```csharp
builder.Services.AddAWSLambdaHosting(LambdaEventSource.HttpApi);
builder.Services.AddDocLensServices(builder.Configuration);
app.UseMiddleware<TenantMiddleware>();
app.MapDocumentEndpoints();
```

## Request Flow

**Intake Lambda** (API Gateway → SQS):

```
API Gateway → TenantMiddleware → DocumentEndpoints
                                    → DocumentExtractionService.EnqueueAsync
                                        └── SQS.SendMessage → 202 Accepted
```

**Worker Lambda** (SQS consumer):

```
SQS → WorkerHandler
         → GuardDuty tag gate (S3:GetObjectTagging)
         → DocumentExtractionService.ExtractAsync
              ├── TextractOcrService      (raw text)
              └── BedrockSemanticAnalysis (structured fields)
         → DynamoDB write (COMPLETED / DUPLICATE / REJECTED)
         → SQS notifications queue (ExtractionCompleted)
```

## Key Services

| Service | Interface | Purpose |
|---|---|---|
| `DocumentExtractionService` | `IDocumentExtractionService` | Intake: enqueues processing job to SQS. Worker: orchestrates GuardDuty gate, OCR, semantic analysis, and DynamoDB write |
| `TextractOcrService` | `IOcrService` | Calls Textract to extract raw text from S3 document (worker Lambda only) |
| `BedrockSemanticAnalysisService` | `ISemanticAnalysisService` | Calls Bedrock/Claude to extract typed fields from raw text (worker Lambda only) |

## Multi-Tenant Rule

Every service that touches data **must** receive `ITenantContext` via constructor injection and include `TenantId` in all storage keys, log statements, and response objects.

| Layer | Key |
|---|---|
| S3 | `{tenantId}/documents/{documentId}/v{versionNumber}.pdf` |
| DynamoDB | `TENANT#{tenantId}` |
| Logs | Structured field `TenantId` on every entry |

## Coding Conventions

- C# records for all DTOs (`ProcessDocumentRequest`, `ExtractionResult`).
- Interface + implementation for every service.
- Services: `Scoped`. AWS SDK clients: `Singleton` (via `AddAWSService<T>`).
- Never instantiate AWS clients directly — always use DI.
- Structured logging with `ILogger<T>` — always include `TenantId`, `DocumentId`, and operation context.

## Adding a New Document Type

1. Add the variant to `Models/DocumentType.cs`.
2. Add a matching `case` in `BedrockSemanticAnalysisService.BuildPrompt` with the relevant extraction fields.

## Adding a New Endpoint

1. Add the route to the relevant file in `Endpoints/` (or create a new one for a new domain).
2. Register it in `MapDocumentEndpoints` (or add a new `Map*Endpoints` extension and call it from `Program.cs`).
3. Create request/response records in `Models/`.
4. Implement the backing service under `Services/`.

## Running Locally

```bash
dotnet build
dotnet test
dotnet run --project src/DocLens.Lambda
```

## Deployment

```bash
# Requires: dotnet tool install -g Amazon.Lambda.Tools
dotnet lambda deploy-function --project-location src/DocLens.Lambda
```

Full CDK-based deployment is defined in `infra/src/DocLens.Infra/` but not yet wired to a pipeline.

## Configuration Reference

All settings live in `appsettings.json` and can be overridden via environment variables in Lambda using the `__` separator (e.g. `Bedrock__ModelId`).

| Section | Key | Default | Description |
|---|---|---|---|
| `Bedrock` | `ModelId` | `anthropic.claude-3-haiku-20240307-v1:0` | Bedrock model ID used for semantic extraction |
| `Bedrock` | `MaxTokens` | `1024` | Maximum tokens in the Bedrock response |
| `Logging.LogLevel` | `Default` | `Information` | Minimum log level for application code |
| `Logging.LogLevel` | `Microsoft.AspNetCore` | `Warning` | Minimum log level for ASP.NET Core internals |

**Overriding in Lambda (AWS Console or Terraform):**

```
Bedrock__ModelId   = anthropic.claude-3-5-sonnet-20241022-v2:0
Bedrock__MaxTokens = 2048
```

Strongly-typed binding is handled by `Options/BedrockOptions.cs`, registered with `ValidateOnStart()` — a misconfigured value causes the Lambda to fail fast on cold start rather than at request time.

## Testing

- Unit tests in `tests/DocLens.Lambda.Tests/`.
- Mock all AWS SDK interfaces with NSubstitute — never make real AWS calls in tests.
- Each service test file mirrors the `Services/` layout.

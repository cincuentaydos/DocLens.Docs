# DocLens.Lambda

Core Lambda function. Receives document processing requests via API Gateway (HTTP API v2), runs OCR, performs semantic analysis through Bedrock, and returns structured data.

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

```
API Gateway → TenantMiddleware → DocumentEndpoints
                                    → DocumentExtractionService
                                        ├── TextractOcrService      (raw text)
                                        └── BedrockSemanticAnalysis (structured fields)
```

## Key Services

| Service | Interface | Purpose |
|---|---|---|
| `DocumentExtractionService` | `IDocumentExtractionService` | Orchestrates OCR + semantic analysis |
| `TextractOcrService` | `IOcrService` | Calls Textract to extract raw text from S3 document |
| `BedrockSemanticAnalysisService` | `ISemanticAnalysisService` | Calls Bedrock/Claude to extract typed fields from raw text |

## Multi-Tenant Rule

Every service that touches data **must** receive `ITenantContext` via constructor injection and include `TenantId` in all storage keys, log statements, and response objects.

| Layer | Key |
|---|---|
| S3 | `{tenantId}/{year}/{month}/{documentId}.pdf` |
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

## Testing

- Unit tests in `tests/DocLens.Lambda.Tests/`.
- Mock all AWS SDK interfaces with NSubstitute — never make real AWS calls in tests.
- Each service test file mirrors the `Services/` layout.

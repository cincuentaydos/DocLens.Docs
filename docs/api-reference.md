# API Reference

All endpoints are served via **Amazon API Gateway (HTTP API v2)** and require a valid Cognito JWT in the `Authorization` header.

## Authentication

Every request must include a Bearer token issued by Amazon Cognito:

```
Authorization: Bearer <id-token>
```

The token must carry the `custom:tenantId` claim. Requests with a missing or empty `tenantId` are rejected by `TenantMiddleware` with `401 Unauthorized` before any handler runs.

---

## Endpoints

### `POST /documents/prepare`

Generates a pre-signed S3 PUT URL for direct document upload. Creates a `PENDING` record in DynamoDB.

**Request**

```http
POST /documents/prepare
Authorization: Bearer <token>
Content-Type: application/json

{
  "documentType": "Invoice"
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `documentType` | string (enum) | Yes | One of: `Invoice`, `Contract`, `Report`, `Cv` |

**Response — `201 Created`**

```json
{
  "documentId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "uploadUrl": "https://s3.amazonaws.com/...",
  "expiresAt": "2026-06-30T15:30:00Z"
}
```

| Field | Description |
|---|---|
| `documentId` | Stable identifier for this document — use in subsequent calls |
| `uploadUrl` | Pre-signed S3 PUT URL. Valid for 15 minutes. Accepts `application/pdf` only |
| `expiresAt` | UTC timestamp when `uploadUrl` expires |

**Error responses**

| Status | Condition |
|---|---|
| `401 Unauthorized` | Missing or invalid JWT, or `tenantId` claim absent |
| `400 Bad Request` | Invalid or missing `documentType` |

---

### `POST /documents/process`

Triggers OCR and semantic extraction for a previously uploaded document. Reads the GuardDuty scan tag from S3 before processing.

**Request**

```http
POST /documents/process
Authorization: Bearer <token>
Content-Type: application/json

{
  "documentId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "s3Key": "tenant-abc/2026/06/3fa85f64-5717-4562-b3fc-2c963f66afa6.pdf",
  "documentType": "Invoice"
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `documentId` | string (UUID) | Yes | Returned by `POST /documents/prepare` |
| `s3Key` | string | Yes | S3 object key of the uploaded file |
| `documentType` | string (enum) | Yes | One of: `Invoice`, `Contract`, `Report`, `Cv` |

!!! danger "S3 key tenant validation"
    The Lambda **must** verify that the `s3Key` prefix matches the `tenantId` resolved from the JWT before reading the object. A client that passes an `s3Key` belonging to another tenant must receive `403 Forbidden`. This check is the last line of defence against cross-tenant data access at the processing layer.

**Response — `200 OK`**

```json
{
  "documentId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "tenantId": "tenant-abc",
  "documentType": "Invoice",
  "fields": {
    "issuer": "Acme Corp",
    "invoice_number": "INV-2026-001",
    "issue_date": "2026-06-01",
    "due_date": "2026-06-30",
    "total_amount": "1500.00",
    "currency": "EUR"
  },
  "processedAt": "2026-06-30T14:22:10Z"
}
```

| Field | Description |
|---|---|
| `documentId` | Echo of the input document ID |
| `tenantId` | Tenant resolved from the JWT — never supplied by the client |
| `documentType` | Echo of the input document type |
| `fields` | Extracted fields as key-value pairs. Keys depend on document type (see below) |
| `processedAt` | UTC timestamp of when extraction completed |

**Error responses**

| Status | Condition |
|---|---|
| `401 Unauthorized` | Missing or invalid JWT, or `tenantId` claim absent |
| `400 Bad Request` | Missing required fields |
| `409 Conflict` | GuardDuty scan not yet complete — client should retry with backoff |
| `422 Unprocessable Entity` | File tagged `THREATS_FOUND` or `UNSCANNABLE` by GuardDuty |

---

## Extracted Fields by Document Type

The `fields` object in the `POST /documents/process` response contains different keys depending on `documentType`. All values are strings. Missing fields are returned as empty strings.

### `Invoice`

| Field | Description |
|---|---|
| `issuer` | Name of the issuing company |
| `recipient` | Name of the recipient company |
| `invoice_number` | Invoice identifier |
| `issue_date` | Date the invoice was issued |
| `due_date` | Payment due date |
| `total_amount` | Total amount due |
| `currency` | Currency code (e.g. `EUR`, `USD`) |
| `line_items_summary` | Free-text summary of line items |

### `Contract`

| Field | Description |
|---|---|
| `parties` | Names of the contracting parties |
| `effective_date` | Date the contract takes effect |
| `expiry_date` | Contract expiry date |
| `governing_law` | Jurisdiction |
| `key_obligations_summary` | Free-text summary of key obligations |

### `Report`

| Field | Description |
|---|---|
| `title` | Report title |
| `author` | Author name(s) |
| `date` | Report date |
| `summary` | Executive summary |
| `key_findings` | Free-text summary of key findings |

### `Cv`

| Field | Description |
|---|---|
| `full_name` | Candidate full name |
| `email` | Email address |
| `phone` | Phone number |
| `current_role` | Current or most recent job title |
| `skills` | Comma-separated skills |
| `education_summary` | Education background summary |
| `experience_summary` | Work experience summary |

---

## Client Integration Flow

```
1. POST /documents/prepare
   → receive { documentId, uploadUrl }

2. PUT {uploadUrl}
   Content-Type: application/pdf
   Body: <file bytes>
   (direct S3 call — not through API Gateway)
   → receive 200 from S3

3. Wait a few seconds for GuardDuty to scan the object

4. POST /documents/process
   → receive extraction result
   → if 409, retry with backoff (scan still running)
```

!!! warning "409 retry behaviour"
    A `409 Conflict` means the GuardDuty scan has not yet completed. Implement an exponential backoff retry — typically the scan finishes within 5–15 seconds. Do not poll more than a few times; if the scan does not complete within a reasonable window, surface an error to the user.

---

## Adding a New Document Type

1. Add the variant to `Models/DocumentType.cs`.
2. Add a matching `case` in `BedrockSemanticAnalysisService.BuildPrompt` listing the fields to extract.
3. Document the new fields in the **Extracted Fields** section above.

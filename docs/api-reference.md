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
  "documentType": "Invoice",
  "documentId": "3fa85f64-5717-4562-b3fc-2c963f66afa6"
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `documentType` | string (enum) | Yes | One of: `Invoice`, `Contract`, `Report`, `Cv` |
| `documentId` | string (UUID) | No | If provided, creates a new version of an existing document. If absent, starts a new document (v1). |

**Response — `201 Created`**

```json
{
  "documentId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "versionNumber": 2,
  "uploadUrl": "https://s3.amazonaws.com/...",
  "expiresAt": "2026-07-01T10:15:00Z"
}
```

| Field | Description |
|---|---|
| `documentId` | Stable identifier for this document across all versions |
| `versionNumber` | Version number for this upload — `1` for new documents, incremented for subsequent versions |
| `uploadUrl` | Pre-signed S3 PUT URL. Valid for 15 minutes. Accepts `application/pdf` only |
| `expiresAt` | UTC timestamp when `uploadUrl` expires |

**Error responses**

| Status | Condition |
|---|---|
| `401 Unauthorized` | Missing or invalid JWT, or `tenantId` claim absent |
| `400 Bad Request` | Invalid or missing `documentType` |
| `403 Forbidden` | `documentId` provided but does not belong to the authenticated tenant |
| `404 Not Found` | `documentId` provided but does not exist |

---

### `POST /documents/process`

Enqueues the document for asynchronous OCR and semantic extraction. Returns `202 Accepted` immediately — the client polls `GET /documents/{documentId}/versions/{versionNumber}` for the result. GuardDuty gate, SHA check, and extraction are handled internally by the worker Lambda.

**Request**

```http
POST /documents/process
Authorization: Bearer <token>
Content-Type: application/json

{
  "documentId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "versionNumber": 2,
  "s3Key": "tenant-abc/documents/3fa85f64-5717-4562-b3fc-2c963f66afa6/v2.pdf",
  "documentType": "Invoice"
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `documentId` | string (UUID) | Yes | Returned by `POST /documents/prepare` |
| `versionNumber` | number | Yes | Returned by `POST /documents/prepare` |
| `s3Key` | string | Yes | S3 object key of the uploaded file — must match `{tenantId}/documents/{documentId}/v{versionNumber}.pdf` |
| `documentType` | string (enum) | Yes | One of: `Invoice`, `Contract`, `Report`, `Cv` |

!!! danger "S3 key tenant validation"
    The Lambda **must** verify that the `s3Key` prefix matches the `tenantId` resolved from the JWT before reading the object. A client that passes an `s3Key` belonging to another tenant must receive `403 Forbidden`. This check is the last line of defence against cross-tenant data access at the processing layer.

**Response — `202 Accepted`**

```json
{
  "documentId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "versionNumber": 2
}
```

The extraction result is not included in this response. Poll `GET /documents/{documentId}/versions/{versionNumber}` until `status` is no longer `PENDING`.

**Error responses**

| Status | Condition |
|---|---|
| `401 Unauthorized` | Missing or invalid JWT, or `tenantId` claim absent |
| `400 Bad Request` | Missing required fields or `s3Key` does not match expected pattern |
| `403 Forbidden` | `s3Key` prefix does not match the authenticated tenant |

---

### `GET /documents/{documentId}/versions`

Lists all versions of a document in reverse chronological order (latest first).

**Request**

```http
GET /documents/3fa85f64-5717-4562-b3fc-2c963f66afa6/versions
Authorization: Bearer <token>
```

**Response — `200 OK`**

```json
{
  "documentId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "documentType": "Invoice",
  "latestVersion": 2,
  "versions": [
    {
      "versionNumber": 2,
      "status": "COMPLETED",
      "sha256": "e3b0c44298fc1c149afb...",
      "processedAt": "2026-07-01T10:20:00Z"
    },
    {
      "versionNumber": 1,
      "status": "COMPLETED",
      "sha256": "a87ff679a2f3e71d9181...",
      "processedAt": "2026-06-15T09:05:00Z"
    }
  ]
}
```

**Error responses**

| Status | Condition |
|---|---|
| `401 Unauthorized` | Missing or invalid JWT |
| `403 Forbidden` | Document does not belong to the authenticated tenant |
| `404 Not Found` | `documentId` does not exist |

---

### `GET /documents/{documentId}/versions/{versionNumber}`

Returns the full extracted fields and diff for a specific version.

**Request**

```http
GET /documents/3fa85f64-5717-4562-b3fc-2c963f66afa6/versions/2
Authorization: Bearer <token>
```

**Response — `200 OK`**

```json
{
  "documentId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "versionNumber": 2,
  "tenantId": "tenant-abc",
  "documentType": "Invoice",
  "status": "COMPLETED",
  "sha256": "e3b0c44298fc1c149afb...",
  "fields": {
    "issuer": "Acme Corp",
    "total_amount": "1800.00"
  },
  "diffFromPrevious": [
    { "op": "replace", "path": "/total_amount", "value": "1800.00" }
  ],
  "processedAt": "2026-07-01T10:20:00Z"
}
```

**Error responses**

| Status | Condition |
|---|---|
| `401 Unauthorized` | Missing or invalid JWT |
| `403 Forbidden` | Document does not belong to the authenticated tenant |
| `404 Not Found` | `documentId` or `versionNumber` does not exist |

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

=== "New document (v1)"

    ```
    1. POST /documents/prepare  { documentType }
       → receive { documentId, versionNumber: 1, uploadUrl }

    2. PUT {uploadUrl}  (direct S3 — bypasses Lambda)
       → 200 OK from S3

    3. POST /documents/process  { documentId, versionNumber: 1, s3Key, documentType }
       → 202 Accepted

    4. poll GET /documents/{documentId}/versions/1
       until status ≠ "PENDING"
       → COMPLETED : fields available
       → REJECTED  : file flagged by GuardDuty
    ```

=== "New version of existing document"

    ```
    1. POST /documents/prepare  { documentType, documentId }
       → receive { documentId, versionNumber: 2, uploadUrl }

    2. PUT {uploadUrl}  (direct S3 — bypasses Lambda)
       → 200 OK from S3

    3. POST /documents/process  { documentId, versionNumber: 2, s3Key, documentType }
       → 202 Accepted

    4. poll GET /documents/{documentId}/versions/2
       until status ≠ "PENDING"
       → COMPLETED  : fields and diffFromPrevious available
       → DUPLICATE  : content unchanged — refer to previous version
       → REJECTED   : file flagged by GuardDuty
    ```

!!! tip "Polling strategy"
    Start polling after 2 seconds. Back off to 5-second intervals after 10 seconds. Cap at 10-second intervals. Surface an error to the user if status remains `PENDING` after 5 minutes — this indicates a worker failure; check the SQS DLQ.

---

## Adding a New Document Type

1. Add the variant to `Models/DocumentType.cs`.
2. Add a matching `case` in `BedrockSemanticAnalysisService.BuildPrompt` listing the fields to extract.
3. Document the new fields in the **Extracted Fields** section above.

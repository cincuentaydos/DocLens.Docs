# Referencia de API

Todos los endpoints se sirven vía **Amazon API Gateway** y requieren un JWT válido de Cognito en el encabezado `Authorization`.

## Autenticación

Cada solicitud debe incluir un token Bearer emitido por Amazon Cognito:

```
Authorization: Bearer <id-token>
```

El token debe llevar el claim `custom:tenantId`. Solicitudes con un `tenantId` ausente o vacío son rechazadas por el middleware de tenant con `401 Unauthorized` antes de que corra cualquier handler.

Además del tenant, cada solicitud resuelve el **tipo de usuario** autenticado (`interno_admin`, `interno`, o `cliente`) y, para usuarios de tipo `cliente`, el `clienteId` al que pertenecen — ver `tenant-onboarding.md`.

---

## Procesos

### `POST /procesos`

Crea un nuevo proceso legal. Solo usuarios internos.

**Solicitud**

```http
POST /procesos
Authorization: Bearer <token>
Content-Type: application/json

{
  "titulo": "Contrato de arrendamiento — Acme Corp",
  "clienteId": "3fa85f64-5717-4562-b3fc-2c963f66afa6"
}
```

**Respuesta — `201 Created`**

```json
{
  "id": "9c858901-8a57-4791-81fe-4c455b099bc9",
  "empresaId": "tenant-abc",
  "clienteId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "titulo": "Contrato de arrendamiento — Acme Corp",
  "estado": "ABIERTO",
  "creadoEn": "2026-09-07T10:00:00Z"
}
```

### `PATCH /procesos/{procesoId}`

Actualiza un proceso existente (título, estado). Solo usuarios internos con permiso sobre el proceso.

### `GET /procesos/{procesoId}`

Consulta el estado y los datos de un proceso. Usuarios internos con permiso, o usuarios de cliente cuyo `clienteId` está asociado al proceso (ver [Permisos](#permisos-y-auditoria)).

**Respuesta — `200 OK`**

```json
{
  "id": "9c858901-8a57-4791-81fe-4c455b099bc9",
  "clienteId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "titulo": "Contrato de arrendamiento — Acme Corp",
  "estado": "ABIERTO",
  "creadoEn": "2026-09-07T10:00:00Z",
  "actualizadoEn": "2026-09-07T10:00:00Z"
}
```

### `GET /procesos`

Lista los procesos visibles para el usuario autenticado — todos los del tenant para usuarios internos; solo los asociados a su `clienteId` para usuarios de cliente.

**Errores comunes**

| Estado | Condición |
|---|---|
| `401 Unauthorized` | JWT ausente o inválido |
| `403 Forbidden` | El usuario no tiene permiso sobre el proceso solicitado |
| `404 Not Found` | El `procesoId` no existe o no pertenece al tenant autenticado |

---

## Clientes

### `POST /clientes`

Crea un nuevo cliente del despacho. Solo usuarios internos admin.

```http
POST /clientes
Authorization: Bearer <token>
Content-Type: application/json

{ "nombre": "Acme Corp" }
```

### `GET /clientes/{clienteId}`

Consulta un cliente y sus procesos asociados.

### `POST /clientes/{clienteId}/procesos/{procesoId}`

Asocia un proceso existente a un cliente.

### `POST /clientes/{clienteId}/usuarios`

Crea un usuario de tipo `cliente`, vinculado a ese `clienteId`, para que pueda consultar sus procesos. Solo usuarios internos admin.

```http
POST /clientes/3fa85f64-5717-4562-b3fc-2c963f66afa6/usuarios
Authorization: Bearer <token>
Content-Type: application/json

{ "email": "contacto@acme.com" }
```

Internamente, este endpoint provisiona el usuario en Cognito con `custom:tenantId` y un claim adicional que lo vincula al `clienteId` — ver `tenant-onboarding.md`.

---

## Documentos

### `POST /documents/prepare`

Genera una URL S3 PUT prefirmada para la subida directa de un documento asociado a un proceso. Crea un registro `PENDING` en Aurora. Ver [ADR-005](adrs/005-upload-strategy.md) para el mecanismo completo.

**Solicitud**

```http
POST /documents/prepare
Authorization: Bearer <token>
Content-Type: application/json

{
  "procesoId": "9c858901-8a57-4791-81fe-4c455b099bc9",
  "tipoDocumento": "contrato",
  "contentType": "application/pdf",
  "documentId": "3fa85f64-5717-4562-b3fc-2c963f66afa6"
}
```

| Campo | Tipo | Requerido | Descripción |
|---|---|---|---|
| `procesoId` | string (UUID) | Sí | Proceso al que pertenece el documento |
| `tipoDocumento` | string | No | Categoría libre (p. ej. "contrato", "escrito", "prueba", "comunicación") |
| `contentType` | string | Sí | MIME type — ver la lista de formatos soportados en [ADR-005](adrs/005-upload-strategy.md#formatos-soportados-lista-permitida) |
| `documentId` | string (UUID) | No | Si se provee, crea una nueva versión de un documento existente |

**Respuesta — `201 Created`**

```json
{
  "documentId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "versionNumber": 1,
  "uploadUrl": "https://s3.amazonaws.com/...",
  "expiresAt": "2026-09-07T10:15:00Z"
}
```

El frontend hace PUT del archivo directamente a `uploadUrl`. **No existe una llamada `/process`** — el procesamiento se dispara automáticamente cuando GuardDuty confirma que el archivo está limpio (ver [ADR-011](adrs/011-document-processing-trigger.md)).

### `GET /procesos/{procesoId}/documentos`

Lista los documentos de un proceso (resumen, última versión de cada uno).

### `GET /documents/{documentId}/versions`

Lista todas las versiones de un documento.

### `GET /documents/{documentId}/versions/{versionNumber}`

Devuelve los campos extraídos y el diff de una versión específica.

**Respuesta — `200 OK`**

```json
{
  "documentId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "versionNumber": 2,
  "procesoId": "9c858901-8a57-4791-81fe-4c455b099bc9",
  "tipoDocumento": "contrato",
  "status": "COMPLETED",
  "sha256": "e3b0c44298fc1c149afb...",
  "fields": {
    "partes": "Acme Corp, Globex S.A.",
    "fecha_vigencia": "2026-01-01",
    "fecha_vencimiento": "2027-01-01"
  },
  "diffFromPrevious": [
    { "op": "replace", "path": "/fecha_vencimiento", "value": "2027-01-01" }
  ],
  "processedAt": "2026-09-07T10:20:00Z"
}
```

**Errores comunes**

| Estado | Condición |
|---|---|
| `401 Unauthorized` | JWT ausente o inválido |
| `400 Bad Request` | `contentType` no soportado, o campos requeridos ausentes |
| `403 Forbidden` | El usuario no tiene permiso sobre el proceso, o el `documentId` no pertenece al tenant autenticado |
| `404 Not Found` | `documentId` o `versionNumber` no existe |

!!! tip "Estrategia de sondeo"
    Empezar a sondear después de 2 segundos. Reducir a intervalos de 5 segundos tras 10 segundos. Tope de 10 segundos. Mostrar un error al usuario si el estado sigue en `PENDING` después de 5 minutos — indica un fallo del worker; revisar la DLQ de SQS.

---

## Usuarios

### `POST /usuarios`

Crea un usuario interno (`interno_admin` o `interno`). Solo usuarios internos admin.

```http
POST /usuarios
Authorization: Bearer <token>
Content-Type: application/json

{ "email": "abogada@despacho.com", "tipo": "interno" }
```

| Campo | Tipo | Requerido | Descripción |
|---|---|---|---|
| `email` | string | Sí | Correo del usuario |
| `tipo` | string (enum) | Sí | `interno_admin` o `interno` |

### `GET /usuarios`

Lista los usuarios internos del tenant. Solo usuarios internos admin.

### `PATCH /usuarios/{usuarioId}`

Actualiza el tipo de un usuario interno (promover/degradar admin) o desactivarlo.

---

## Permisos y Auditoría

### `POST /procesos/{procesoId}/permisos`

Otorga a un usuario (interno o de cliente) acceso a un proceso específico.

```http
POST /procesos/9c858901-8a57-4791-81fe-4c455b099bc9/permisos
Authorization: Bearer <token>
Content-Type: application/json

{ "usuarioId": "...", "nivel": "lectura" }
```

### `GET /audit-log`

Devuelve el registro de auditoría de accesos — quién consultó qué proceso/documento y cuándo. Solo usuarios internos admin.

```json
{
  "entradas": [
    {
      "usuarioId": "...",
      "accion": "CONSULTA_PROCESO",
      "recurso": "proceso:9c858901-8a57-4791-81fe-4c455b099bc9",
      "resultado": "PERMITIDO",
      "creadoEn": "2026-09-07T09:00:00Z"
    }
  ]
}
```

---

## Inteligencia Artificial

Ambos endpoints siguen el flujo explícito descrito en [ADR-012](adrs/012-no-autonomous-agent.md): autorización → retrieval en Bedrock Knowledge Bases (filtrado por `empresa_id` + `procesoId`) → generación con Claude → respuesta con fuentes citadas.

### `POST /procesos/{procesoId}/consultas`

Responde una pregunta en el contexto de un proceso.

```http
POST /procesos/9c858901-8a57-4791-81fe-4c455b099bc9/consultas
Authorization: Bearer <token>
Content-Type: application/json

{ "pregunta": "¿Cuál es la fecha de vencimiento del contrato de arrendamiento?" }
```

**Respuesta — `200 OK`**

```json
{
  "respuesta": "El contrato de arrendamiento vence el 2027-01-01.",
  "fuentes": [
    { "documentId": "3fa85f64-5717-4562-b3fc-2c963f66afa6", "versionNumber": 2 }
  ]
}
```

### `POST /procesos/{procesoId}/borradores`

Genera un primer borrador de documento (comunicación, escrito) a partir del contexto del proceso.

```http
POST /procesos/9c858901-8a57-4791-81fe-4c455b099bc9/borradores
Authorization: Bearer <token>
Content-Type: application/json

{ "tipo": "carta_requerimiento", "instrucciones": "Solicitar el pago pendiente antes del 30 de septiembre" }
```

**Respuesta — `200 OK`**

```json
{
  "borrador": "Estimados señores de Globex S.A. ...",
  "fuentes": [
    { "documentId": "3fa85f64-5717-4562-b3fc-2c963f66afa6", "versionNumber": 2 }
  ]
}
```

**Errores comunes**

| Estado | Condición |
|---|---|
| `401 Unauthorized` | JWT ausente o inválido |
| `403 Forbidden` | El usuario no tiene permiso sobre el proceso consultado |
| `422 Unprocessable Entity` | La base de conocimiento no tiene contenido indexado para este proceso |

---

## Flujo de Integración del Cliente

=== "Documento nuevo (v1)"

    ```
    1. POST /documents/prepare  { procesoId, contentType, tipoDocumento }
       → recibe { documentId, versionNumber: 1, uploadUrl }

    2. PUT {uploadUrl}  (directo a S3 — evita Lambda)
       → 200 OK de S3

    3. GuardDuty escanea → EventBridge → SQS → Lambda Processor
       (sin llamada adicional del cliente — ver ADR-011)

    4. sondear GET /documents/{documentId}/versions/1
       hasta status ≠ "PENDING"
       → COMPLETED : campos disponibles
       → REJECTED  : archivo marcado por GuardDuty
    ```

=== "Nueva versión de un documento existente"

    ```
    1. POST /documents/prepare  { procesoId, contentType, documentId }
       → recibe { documentId, versionNumber: 2, uploadUrl }

    2. PUT {uploadUrl}  (directo a S3 — evita Lambda)
       → 200 OK de S3

    3. GuardDuty escanea → EventBridge → SQS → Lambda Processor

    4. sondear GET /documents/{documentId}/versions/2
       hasta status ≠ "PENDING"
       → COMPLETED  : campos y diffFromPrevious disponibles
       → DUPLICATE  : contenido sin cambios — referirse a la versión anterior
       → REJECTED   : archivo marcado por GuardDuty
    ```

---

## Agregar un Nuevo Tipo de Documento

`tipoDocumento` es un campo de texto libre en el dominio actual (no un enum cerrado) — no requiere cambios de código para agregar una nueva categoría. Si en el futuro se necesita una plantilla de extracción específica por tipo, el prompt de `ISemanticAnalysisService` puede ramificarse por `tipoDocumento`; hoy el prompt es único y suficientemente general para el dominio legal.

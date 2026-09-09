# ADR-006 — Procesamiento Síncrono vs. Asíncrono de la Extracción

**Estado:** Superada por [ADR-011](011-document-processing-trigger.md) — el modelo de disparo por llamada explícita del cliente (`POST /documents/process`) y el sondeo del tag de GuardDuty con reintentos, descritos más abajo, fueron reemplazados por un disparo dirigido por eventos (GuardDuty → EventBridge → SQS). La decisión de fondo de esta ADR —procesar de forma asíncrona en lugar de síncrona— sigue siendo válida y es la base sobre la que se construye ADR-011; se conserva este documento como registro histórico del razonamiento original.

---

## Decisión original

**Procesar los documentos de forma asíncrona mediante una Lambda worker impulsada por SQS.** `POST /documents/process` devolvía `202 Accepted` de inmediato. Una Lambda de procesamiento separada consumía la cola SQS y ejecutaba la extracción con Textract + Bedrock. El cliente sondeaba `GET /documents/{documentId}/versions/{versionNumber}` hasta que el estado dejaba de ser `PENDING`.

---

## Contexto

DocLens recibía solicitudes de procesamiento de documentos vía `POST /documents/process`. Una vez que el archivo estaba en S3 y había pasado el escaneo de malware, debían completarse dos llamadas a servicios externos antes de poder devolver datos estructurados: OCR (Textract) y análisis semántico (Bedrock). Había dos formas arquitectónicamente distintas de manejar ese trabajo: de forma síncrona dentro de la misma invocación Lambda que recibía la solicitud, o de forma asíncrona vía una cola de trabajos que desacoplara la recepción del procesamiento.

El diseño inicial optaba por el procesamiento síncrono por simplicidad. Esta decisión se revisó porque el **timeout duro de 29 segundos de API Gateway HTTP API v2** no puede configurarse ni extenderse — es un límite impuesto por AWS. Textract sobre documentos escaneados de varias páginas, combinado con la inferencia de Bedrock, se acercaba o superaba regularmente ese límite. Un `504 Gateway Timeout` en esta capa significaba que el resultado de la extracción se perdía silenciosamente, sin posibilidad de reintento a nivel de infraestructura.

---

## Opciones consideradas

### Opción 1 — Síncrono (en línea) *(enfoque inicial — descartado)*

Textract y Bedrock se invocaban secuencialmente dentro de la misma ejecución Lambda que manejaba `POST /documents/process`. La respuesta HTTP se mantenía abierta hasta que ambas llamadas retornaban.

```
POST /documents/process
  → Lambda invocada
  → Textract (OCR)      ~1–5s típico, más en documentos escaneados multi-página
  → Bedrock (Claude)    ~2–8s típico
  → HTTP 200 devuelto
```

**Fortalezas**

- Integración simple del cliente — una solicitud, una respuesta, sin sondeo.
- Backend simple — sin cola, sin registro de trabajo, sin Lambda worker separada.
- Manejo de errores directo — los fallos aparecían de inmediato como errores HTTP.
- Fácil de observar y depurar — una sola traza cubría toda la operación.

**Debilidades**

- **Timeout de API Gateway:** HTTP API v2 tiene un timeout duro de integración de 29 segundos. Si Textract + Bedrock juntos lo superaban, el cliente recibía un `504 Gateway Timeout` y el resultado se perdía. Documentos escaneados multi-página incumplían este límite regularmente.
- **Sin aislamiento de reintentos:** si la Lambda fallaba a mitad de la extracción, el cliente recibía un 5xx sin resultado parcial y debía reintentar la operación completa.
- **Techo de throughput:** una sola instancia Lambda manejaba un documento a la vez. Alta concurrencia requería escalado de Lambda, lo que añadía cold starts.
- **Sondeo de GuardDuty trasladado al cliente:** el cliente debía implementar backoff exponencial ante `409 Conflict` mientras el escaneo se completaba — filtrando un detalle de infraestructura hacia el frontend.

---

### Opción 2 — Asíncrono (dirigido por eventos, SQS) *(elegida en su momento — ver nota de superación arriba)*

`POST /documents/process` encolaba un trabajo y devolvía `202 Accepted` de inmediato. Una Lambda worker separada consumía la cola SQS y ejecutaba la compuerta de GuardDuty + verificación de SHA + Textract + Bedrock. El cliente sondeaba el endpoint de estado de versión existente.

```
POST /documents/process
  → Lambda de intake encola mensaje en SQS
  → HTTP 202 Accepted

Escaneo de S3 + GuardDuty (manejado internamente por el worker)
SQS → Lambda Worker
  → Compuerta de GuardDuty (reintento interno si el tag está ausente)
  → Verificación de SHA-256
  → Textract + Bedrock
  → escribe COMPLETED / DUPLICATE / REJECTED en la base de datos

GET /documents/{documentId}/versions/{versionNumber}
  → { status: "PENDING" | "COMPLETED" | "DUPLICATE" | "REJECTED", fields, diffFromPrevious }
```

**Fortalezas**

- **Sin restricción de timeout de API Gateway** — el 202 se devolvía antes de que empezara cualquier extracción.
- **GuardDuty manejado internamente** — la Lambda worker reintentaba la verificación del tag internamente; el cliente nunca veía un `409`.
- **Aislamiento de reintentos** — la DLQ de SQS capturaba fallos sin impacto en el cliente; los documentos fallidos no se perdían silenciosamente.
- **Backpressure natural** — SQS amortiguaba picos de tráfico sin descartar solicitudes.
- **Habilitaba trabajos asíncronos de Textract** (`StartDocumentTextDetection`) para documentos escaneados multi-página sin ningún cambio arquitectónico.
- **Lambda worker dimensionada de forma independiente** — más memoria y margen de timeout para cargas pesadas sin afectar la Lambda de intake.
- **Cliente más simple** — el frontend subía el archivo, disparaba el procesamiento, y luego sondeaba un único endpoint de estado.

**Debilidades**

- Infraestructura ligeramente más compleja: cola SQS, DLQ, Lambda worker, cableado de IAM.
- Más difícil de depurar — una extracción fallida debía trazarse a través de dos invocaciones Lambda.
- Añadía latencia para documentos rápidos típicos (overhead de sondeo de cola, normalmente < 1s).

---

### Opción 3 — Completamente dirigido por eventos: S3 → SQS directamente *(propuesta en discusión en su momento)*

Esta opción, planteada originalmente al revisar ADR-005, es la que terminó formalizándose — con GuardDuty y EventBridge como origen del evento en lugar de una notificación `S3:ObjectCreated` — en **[ADR-011](011-document-processing-trigger.md)**. Ver esa ADR para el enfoque final.

---

## Comparación

| Dimensión | Síncrono | Asíncrono (elegido en su momento) | Dirigido por eventos (formalizado en ADR-011) |
|---|---|---|---|
| Complejidad del cliente | Baja (una solicitud) | Baja (202 + sondeo de estado) | La más baja (solo subida + sondeo) |
| Timeout de API Gateway | Límite duro de 29s — inviable para PDFs escaneados | No aplica | No aplica |
| Sondeo de GuardDuty | Del lado del cliente (409 + backoff) | Interno a la Lambda worker | Resuelto por la regla de EventBridge — no hay sondeo |
| PDFs escaneados grandes | Riesgo de 504 | Totalmente soportado | Totalmente soportado |
| Aislamiento de reintentos | Ninguno | DLQ de SQS | DLQ de SQS |

---

## Consecuencias (históricas)

- **API Reference:** `POST /documents/process` devolvía `202 Accepted` en lugar de `200 OK` con campos — este endpoint fue eliminado en ADR-011.
- **Timeout de Lambda:** la Lambda de intake podía reducirse a ~10 segundos (solo escritura en cola). La Lambda worker se configuraba a **5 minutos** para acomodar trabajos largos de Textract.

## Nota de superación

Ver **[ADR-011 — Disparo del Procesamiento de Documentos](011-document-processing-trigger.md)** para el mecanismo actual: GuardDuty → EventBridge → SQS, sin endpoint `/process` ni sondeo de tag.

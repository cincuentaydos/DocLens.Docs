# ADR-011 — Disparo del Procesamiento de Documentos

**Estado:** Aceptada (2026-09-07) — Supera a [ADR-006](006-sync-vs-async-processing.md) y sustituye el mecanismo de compuerta descrito en la sección "Enfoque elegido" de [ADR-004](004-malware-scanning.md).

---

## Decisión

El procesamiento de un documento se dispara automáticamente cuando **Amazon GuardDuty Malware Protection completa el escaneo**, sin que el cliente llame a ningún endpoint de tipo `/process`.

```
GuardDuty (resultado del escaneo) → regla de EventBridge (filtra solo resultado limpio) → SQS + DLQ → Lambda Processor
```

La Lambda Processor ya no necesita leer el tag `GuardDutyMalwareScanStatus` por sondeo con reintentos — la regla de EventBridge solo reenvía a SQS cuando el resultado del escaneo es el adecuado. El endpoint `POST /documents/process` desaparece del flujo del cliente.

---

## Contexto

[ADR-006](006-sync-vs-async-processing.md) decidió el procesamiento asíncrono vía SQS, pero mantenía un endpoint `POST /documents/process` llamado explícitamente por el cliente después del `PUT` a S3, y dejaba que la Lambda worker sondeara el tag de GuardDuty con reintentos vía el *visibility timeout* de SQS cuando el escaneo aún no había terminado. Esa misma ADR ya dejaba anotada, como "Opción 3 — bajo discusión", la posibilidad de un disparo completamente dirigido por eventos.

Al redefinir la arquitectura V1, se formalizó ese disparo por eventos, pero usando **GuardDuty + EventBridge** como origen del evento (no una notificación `S3:ObjectCreated`), ya que GuardDuty Malware Protection para S3 publica el resultado del escaneo como evento en EventBridge de forma nativa.

## Opciones consideradas

### Opción 1 — Endpoint `/process` + sondeo de tag en la Lambda worker (ADR-006 original)

**Fortalezas:** señal explícita del cliente de "empezar a procesar"; simple de trazar en un solo flujo lógico.

**Debilidades:** el cliente debe orquestar una llamada adicional después del `PUT`; el caso de "tag ausente" (escaneo aún en curso) es manejado con reintentos de SQS, añadiendo latencia y complejidad al Lambda worker; existe una ventana en la que un `PENDING` puede quedar huérfano si el cliente nunca llama a `/process`.

### Opción 2 — Notificación `S3:ObjectCreated` → SQS directamente

Disparar el procesamiento en cuanto el `PUT` a S3 se completa, sin pasar por GuardDuty como filtro explícito en el evento de disparo.

**Debilidades:** el escaneo de GuardDuty apenas ha comenzado cuando el evento de S3 se dispara — el caso "tag ausente" pasa a ser el caso común en lugar de la excepción, empujando de vuelta el problema de reintentos hacia la Lambda worker.

### Opción 3 — GuardDuty → EventBridge → SQS (elegida)

**Fortalezas:**

- El evento que dispara el procesamiento es exactamente "el escaneo terminó con resultado limpio" — no hay ventana de "tag ausente" que gestionar por reintentos: la regla de EventBridge simplemente no reenvía nada hasta que el resultado exista.
- Elimina el endpoint `/process` y la Lambda de intake asociada — menos código, menos superficie de API.
- Elimina la ventana de `PENDING` huérfano — no existe un paso intermedio donde la subida se complete pero el procesamiento nunca se dispare.
- El filtrado (solo continuar con resultado limpio) ocurre en la propia regla de EventBridge, no dentro de la lógica de negocio de la Lambda Processor.

**Debilidades:**

- Un componente más en la infraestructura (regla de EventBridge) a definir en Terraform.
- Depurar el flujo requiere revisar la regla de EventBridge además de la cola SQS y la Lambda — la trazabilidad end-to-end pasa por un componente administrado adicional.

---

## Comparación

| Dimensión | Endpoint `/process` + sondeo (ADR-006) | S3:ObjectCreated directo | GuardDuty → EventBridge (elegida) |
|---|---|---|---|
| Llamada explícita del cliente | Sí | No | No |
| Caso "escaneo no terminado" | Común, gestionado con reintentos SQS | Muy común | No ocurre — el evento solo existe cuando el escaneo terminó |
| Riesgo de `PENDING` huérfano | Sí | No | No |
| Lambda de intake / endpoint `/process` | Sí | No | No |
| Complejidad de infraestructura | Media | Media | Media (regla de EventBridge) |

---

## Enfoque elegido

```
1. Frontend: POST /documents/prepare → { documentId, versionNumber, uploadUrl }
2. Frontend: PUT {uploadUrl} → 200 OK (directo a S3)
3. S3: versionado, cifrado, objeto original persistido
4. GuardDuty escanea el objeto
5. Regla de EventBridge — filtra únicamente el resultado "limpio"
6. SQS + DLQ — desacopla el evento de EventBridge del procesamiento
7. Lambda Processor consume el mensaje → detecta formato → extrae texto (ver ADR-003) → análisis semántico (Bedrock) → escribe resultado en Aurora
8. Frontend: sondea GET /documents/{documentId}/versions/{versionNumber} hasta que el estado ya no sea PENDING
```

Un resultado `THREATS_FOUND` o `UNSCANNABLE` de GuardDuty no genera evento hacia la cola de procesamiento — la regla de EventBridge lo filtra. El registro correspondiente se marca como `REJECTED` mediante un flujo separado (evento de GuardDuty con resultado no limpio → función ligera que solo actualiza el estado en Aurora), sin pasar por la Lambda Processor.

## Consecuencias

- **[ADR-006](006-sync-vs-async-processing.md):** superada en su totalidad — el endpoint `/process`, la Lambda de intake y el modelo de sondeo de tag ya no aplican.
- **[ADR-004](004-malware-scanning.md):** el mecanismo de compuerta (tag-polling en la Lambda worker) queda superado por esta ADR; la decisión de usar GuardDuty Malware Protection como escáner no cambia.
- **API Reference:** se elimina `POST /documents/process` de la superficie pública de la API.
- **Terraform:** se añade la regla de EventBridge y su política de destino hacia SQS; se elimina la Lambda de intake dedicada a `/process`.
- **IAM:** la Lambda Processor ya no necesita `s3:GetObjectTagging` para sondeo — el filtrado ocurre antes, en EventBridge.
- **Observabilidad:** las trazas de X-Ray deben propagarse desde el evento de EventBridge hacia el mensaje de SQS y la invocación de la Lambda Processor para mantener un trace correlacionado.

## Preguntas abiertas

- ¿Qué componente actualiza el estado a `REJECTED` cuando GuardDuty reporta `THREATS_FOUND`/`UNSCANNABLE` — una regla de EventBridge adicional apuntando a una función ligera, o un manejador dentro de la misma Lambda Processor con una ruta de entrada distinta?
- ¿Cuál es la política de reintentos y el tiempo de vida de mensajes en la DLQ antes de requerir intervención manual?

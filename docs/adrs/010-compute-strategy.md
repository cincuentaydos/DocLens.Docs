# ADR-010 — Estrategia de Cómputo

**Estado:** Aceptada (2026-09-07)

---

## Decisión

Usar **Amazon API Gateway + AWS Lambda** como capa de cómputo del backend para la V1, en lugar de un servicio de contenedores siempre activo (AWS App Runner o Amazon ECS).

---

## Contexto

La aplicación debe estar disponible 24/7, pero eso no implica que se necesite cómputo activo 24/7: la carga de la V1 es mayormente *request-driven* (peticiones cortas de usuarios/clientes) y las tareas largas (OCR, extracción semántica, ingesta a la base de conocimiento) son asíncronas por diseño (ver [ADR-011](011-document-processing-trigger.md)).

## Opciones consideradas

### Opción 1 — AWS App Runner

**Fortalezas:** despliegue simple de una API containerizada, escalado automático, sin gestionar clusters.

**Debilidades:** se revisó su disponibilidad/SLA y no cumple los requisitos de la aplicación para V1. Cómputo permanentemente activo cuando la carga real no lo justifica — costo innecesario para un patrón request-driven.

### Opción 2 — Amazon ECS (Express Mode)

Evaluado como sustituto de App Runner para una API containerizada.

**Fortalezas:** más control que App Runner sobre la configuración del contenedor; útil si aparecen cargas sostenidas o conexiones persistentes (p. ej. WebSockets).

**Debilidades:** la carga de la V1 no tiene tareas sostenidas ni conexiones persistentes — es principalmente request-driven, con trabajo largo ya delegado a colas y Lambdas asíncronas. Introduce complejidad operativa (definición de tareas, servicio, autoescalado del clúster) sin un beneficio claro para el patrón de carga actual.

### Opción 3 — API Gateway + AWS Lambda (elegida)

**Fortalezas:**

- Cómputo bajo demanda — sin pagar por capacidad ociosa entre picos de tráfico.
- Encaja naturalmente con el patrón ya elegido para el procesamiento asíncrono (SQS + Lambda worker, ver [ADR-011](011-document-processing-trigger.md)).
- Escalado automático por invocación, sin gestión de clúster ni de instancias.
- Integración directa con Cognito (autorización de API Gateway) e IAM por función.

**Debilidades:**

- Timeout de integración de API Gateway HTTP API (límite duro) — mitigado ya que el procesamiento largo ocurre fuera del ciclo de petición/respuesta (ver [ADR-011](011-document-processing-trigger.md)).
- Cold starts en Lambdas poco usadas — aceptable para el volumen esperado en V1.

---

## Comparación

| Dimensión | App Runner | ECS Express Mode | API Gateway + Lambda (elegida) |
|---|---|---|---|
| Cómputo permanente | Sí | Sí (o casi) | No — bajo demanda |
| Encaje con carga request-driven | Medio | Medio | Alto |
| Complejidad operativa | Baja | Media-alta | Baja |
| Costo en baja carga | Fijo | Fijo/variable | Proporcional al uso |
| Conexiones persistentes / WebSockets | Posible | Posible | No nativo |
| Cumple disponibilidad/SLA revisado | No | No evaluado como necesario | Sí |

---

## Enfoque elegido

Backend expuesto vía API Gateway, con AWS Lambda ejecutando la lógica de usuarios, clientes, procesos, permisos, documentos, URLs prefirmadas e IA bajo demanda. El trabajo largo (OCR, extracción, ingesta a la base de conocimiento) se delega a una Lambda worker disparada de forma asíncrona (ver [ADR-011](011-document-processing-trigger.md)), evitando cualquier necesidad de cómputo permanente.

## Consecuencias

- **Terraform:** API Gateway HTTP API + funciones Lambda, sin recursos de ECS/App Runner en la V1.
- **Revisitar si:** aparecen cargas sostenidas (streaming, WebSockets) o conexiones persistentes que Lambda no maneje bien — en ese caso, ECS vuelve a evaluarse.

## Preguntas abiertas

- ¿En qué punto de crecimiento de tráfico conviene reevaluar ECS para el backend síncrono?

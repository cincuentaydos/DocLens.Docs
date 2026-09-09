# ADR-012 — Sin Agente Autónomo en V1

**Estado:** Aceptada (2026-09-07)

---

## Decisión

La V1 **no implementa un agente autónomo de IA**. Las funcionalidades de inteligencia artificial siguen un flujo explícito, controlado íntegramente por el backend, sin selección autónoma de herramientas ni pasos de razonamiento fuera del control de la aplicación.

---

## Contexto

El sistema usa IA para tres funciones: consultar información sobre un proceso, redactar borradores de documentos a partir del contexto de un proceso, y extraer información de documentos para construir la base de conocimiento. Existe la tentación arquitectónica de envolver esto en un "agente" que decida por sí mismo qué herramientas invocar y en qué orden. Para V1 se decidió explícitamente no hacerlo.

## Opciones consideradas

### Opción 1 — Agente autónomo

Un agente con acceso a herramientas (buscar en la base de conocimiento, leer un documento, redactar) decide de forma autónoma qué pasos ejecutar para responder una consulta.

**Fortalezas:** flexibilidad ante consultas complejas o multi-paso no anticipadas explícitamente.

**Debilidades:** menor previsibilidad y trazabilidad — más difícil garantizar que cada respuesta respeta la autorización del usuario antes de tocar datos de un proceso. Mayor superficie para alucinación y para comportamientos no verificables. El alcance actual (consulta de contexto, redacción de borradores, extracción) no requiere selección autónoma de herramientas.

### Opción 2 — Flujo explícito orquestado por el backend (elegida)

**Fortalezas:**

- Cada paso es determinista y auditable: autorización → recuperación con filtro de metadatos → generación con Claude → respuesta con fuentes citadas.
- Encaja directamente con el requisito de trazabilidad: identificar las fuentes usadas y facilitar la revisión humana.
- Reduce la superficie de alucinación: Claude solo recibe los fragmentos ya autorizados y recuperados, no decide qué buscar por su cuenta.
- Más simple de probar y de auditar mediante el sistema de benchmark de calidad.

**Debilidades:** menos flexible ante consultas que requerirían múltiples pasos de búsqueda encadenados — no es un problema conocido hoy.

---

## Enfoque elegido

```
Consulta del usuario
      ↓
Backend valida permiso del usuario sobre el proceso solicitado
      ↓
Retrieve en Amazon Bedrock Knowledge Bases,
   filtrado por metadatos (empresa_id + proceso_id)
      ↓
Fragmentos recuperados → prompt a Claude (Amazon Bedrock)
      ↓
Respuesta o borrador + fuentes utilizadas
```

El código de orquestación vive dentro del backend (la Lambda de la API). No existe un servicio independiente llamado "agente". Las llamadas a Bedrock Knowledge Bases y a Claude son pasos explícitos de esta orquestación, no decisiones autónomas de un modelo.

## Consecuencias

- **API Reference:** los endpoints de IA (consulta, redacción de borrador) siempre devuelven las fuentes usadas junto con la respuesta.
- **Multi-tenancy:** el filtro de metadatos en el `Retrieve` de Bedrock Knowledge Bases debe incluir siempre `empresa_id` (tenant) y, cuando aplique, `proceso_id` — nunca es opcional (ver `architecture.md` → Multi-Tenancy).
- **Benchmark:** el sistema de auditoría de calidad puede evaluar cada paso de forma aislada (calidad de retrieval, calidad de la generación) precisamente porque el flujo es explícito y no una caja negra de agente.

## Disparadores de revisión

- Necesidad de una demostración end-to-end que requiera encadenar múltiples herramientas de forma dinámica.
- Aparición de requisitos que exijan que el sistema seleccione y ejecute herramientas de forma autónoma (por ejemplo, decidir por sí mismo entre varias fuentes de datos externas).

## Preguntas abiertas

- ¿Qué formato exacto tiene la citación de fuentes en la respuesta (referencia a `documento_id` + rango de texto, o solo `documento_id`)?

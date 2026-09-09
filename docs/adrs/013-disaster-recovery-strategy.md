# ADR-013 — Estrategia de Recuperación ante Desastres (V1)

**Estado:** Aceptada (2026-09-07) — Formaliza y supera la mención de `eu-west-2` como "consideración de failover" en `index.md` y `overview.md`.

---

## Decisión

La V1 opera en una **única región, `eu-west-1` (Irlanda)**, sin despliegue activo multi-región. La recuperación ante fallos se apoya en mecanismos dentro de la misma región: versionado y protección de documentos en S3, backups automáticos y Point-in-Time Recovery de Aurora, colas de mensajes muertos (DLQ) para trabajos no procesables, observabilidad vía CloudWatch, e infraestructura completamente reproducible mediante Terraform.

---

## Contexto

Versiones anteriores de la documentación mencionaban `eu-west-2` (Londres) como "consideración de failover" sin que existiera una decisión formal ni una implementación asociada. Al definir la arquitectura V1, se evaluó explícitamente si un despliegue activo multi-región era necesario para el lanzamiento inicial.

## Opciones consideradas

### Opción 1 — Despliegue activo multi-región (`eu-west-1` + `eu-west-2`)

**Fortalezas:** mayor disponibilidad ante una caída regional completa de AWS; menor latencia para usuarios distribuidos geográficamente.

**Debilidades:** complejidad y costo adicionales significativos — replicación de Aurora entre regiones, sincronización de S3, enrutamiento de failover en Route 53, duplicación de Bedrock Knowledge Bases. No hay un requisito de negocio actual (SLA, contrato, regulación) que lo justifique para un despacho de abogados en fase de adopción inicial de la plataforma.

### Opción 2 — Región única con recuperación intra-región robusta (elegida)

**Fortalezas:**

- Complejidad y costo mucho menores — un solo cluster de Aurora, un solo conjunto de buckets S3, sin lógica de failover entre regiones.
- Los mecanismos nativos de la región (versionado S3, backups + PITR de Aurora, DLQ, Terraform reproducible) ya cubren los escenarios de fallo más probables: corrupción de datos, fallo de un job de procesamiento, error de despliegue.
- Terraform permite reconstruir la infraestructura completa en otra región si fuera necesario, sin que eso implique mantener una segunda región activa permanentemente.

**Debilidades:** no protege ante una caída completa y prolongada de la región `eu-west-1`. Aceptado como riesgo residual para V1.

---

## Comparación

| Dimensión | Multi-región activa | Región única (elegida) |
|---|---|---|
| Costo de infraestructura | Alto (duplicado) | Bajo |
| Complejidad operativa | Alta | Baja |
| Protección ante caída regional completa | Sí | No (riesgo residual aceptado) |
| Protección ante corrupción de datos / fallo de job | Igual en ambos casos (backups, DLQ) | Igual en ambos casos (backups, DLQ) |
| Tiempo de recuperación tras incidente regional | Bajo (failover automático) | Alto (requiere reconstrucción vía Terraform) |

---

## Enfoque elegido

| Mecanismo | Rol |
|---|---|
| Versionado y cifrado de S3 | Protege el documento original ante sobrescritura o eliminación accidental |
| Backups automáticos + Point-in-Time Recovery de Aurora | Permite restaurar el estado operativo a un punto anterior |
| SQS DLQ | Captura trabajos de procesamiento que no pueden completarse, evitando pérdida silenciosa |
| CloudWatch | Observabilidad y alarmas para detectar degradación antes de que se convierta en incidente |
| Terraform | Infraestructura completamente reproducible — permite reconstruir el entorno en la misma u otra región si fuera necesario |

Un despliegue activo multi-región queda **fuera del alcance inicial** por la complejidad y el costo adicional que implica, y se evaluará únicamente si futuros requisitos de disponibilidad del negocio lo justifican.

## Consecuencias

- **Terraform:** toda la infraestructura se define parametrizada por región, de forma que una futura expansión multi-región no requiera reescribir los módulos desde cero — solo instanciarlos en una región adicional.
- **`index.md` / `overview.md`:** se elimina la fila de `eu-west-2` como "failover consideration"; la tabla de regiones pasa a reflejar únicamente `eu-west-1` como región primaria, con referencia a esta ADR.

## Disparadores de revisión

- Requisitos contractuales o regulatorios de disponibilidad que exijan tolerancia a fallo regional completo.
- Crecimiento de la base de clientes hacia geografías donde la latencia a `eu-west-1` se vuelva un problema de producto.

## Preguntas abiertas

- ¿Cuál es el RTO/RPO objetivo aceptado por el negocio para un incidente regional completo, dado que hoy implicaría reconstrucción manual vía Terraform?

# ADR-008 — Estrategia de Almacenamiento de Datos

**Estado:** Aceptada (2026-09-07)

---

## Decisión

Usar **Amazon Aurora PostgreSQL Serverless v2 con la extensión `pgvector`** como único motor de datos: almacena tanto los datos operativos relacionales (empresas, usuarios, clientes, procesos, documentos, permisos, auditoría) como el vector store que utiliza Amazon Bedrock Knowledge Bases para RAG, en un esquema dedicado dentro de la misma instancia.

Esto reemplaza dos piezas que aparecían en el diseño anterior:

- **Amazon DynamoDB** como base de datos operativa.
- **OpenSearch Serverless** como vector store implícito de Bedrock Knowledge Bases (ver [ADR-001](001-rag-strategy.md)).

---

## Contexto

El diseño original de DocLens usaba DynamoDB para todos los datos operativos (documentos, versiones, tenants) y dejaba que Bedrock Knowledge Bases gestionara su propio vector store (típicamente OpenSearch Serverless) de forma opaca. Con la redefinición del proyecto hacia un dominio de gestión de casos legales (Procesos, Clientes, Documentos, Usuarios, permisos, auditoría), el modelo de datos ganó relaciones más ricas —permisos por usuario/proceso, asociación cliente-proceso, auditoría de accesos— que encajan mejor en un modelo relacional que en el modelo de acceso por clave-valor de DynamoDB.

## Opciones consideradas

### Opción 1 — DynamoDB (enfoque anterior)

**Fortalezas:** escalado automático, sin gestión de servidores, patrón ya usado en el diseño anterior.

**Debilidades:** modelar permisos por usuario/proceso, asociaciones cliente-proceso y consultas de auditoría requiere un diseño de claves cada vez más artificial (GSIs adicionales, ítems duplicados). No resuelve el vector store de la KB — sigue dependiendo de un segundo servicio (OpenSearch Serverless) gestionado por separado.

### Opción 2 — Aurora PostgreSQL Serverless v2 + pgvector (elegida)

**Fortalezas:**

- Un solo motor para datos relacionales y vector store — menos infraestructura que operar y menos superficie de fallo.
- Modelo relacional natural para procesos, clientes, documentos, usuarios y permisos, con relaciones many-to-many e integridad referencial real.
- `pgvector` es un motor de vector store soportado directamente por Amazon Bedrock Knowledge Bases como destino de la base de conocimiento.
- Serverless v2 escala automáticamente según la carga, sin aprovisionar capacidad fija.
- SQL estándar simplifica auditoría, reportes y el sistema de benchmark de calidad de la IA.

**Debilidades:**

- Requiere gestionar un esquema de base de datos y migraciones versionadas (ver el proceso de release en `architecture.md`).
- Menor throughput de escritura bruto que DynamoDB en cargas extremadamente altas — no es una preocupación al volumen esperado de un despacho de abogados.

### Opción 3 — Vector store separado (p. ej. OpenSearch Serverless) + base relacional aparte

**Fortalezas:** cada motor optimizado para su carga específica.

**Debilidades:** dos servicios a operar, dos modelos de costo, sincronización adicional entre el store operativo y el vector store. No aporta ventaja clara frente a la Opción 2 dado que `pgvector` ya es un destino soportado por Bedrock Knowledge Bases.

---

## Comparación

| Dimensión | DynamoDB | Aurora + pgvector (elegida) | Vector store separado |
|---|---|---|---|
| Modelo de datos | Clave-valor | Relacional | Mixto |
| Relaciones (permisos, cliente↔proceso) | Requiere diseño de claves artificial | Natural (foreign keys) | Natural en la parte relacional |
| Vector store para Bedrock KB | No aplica (requiere otro servicio) | Nativo (`pgvector`) | Servicio separado |
| Número de motores a operar | 2 (DynamoDB + vector store de la KB) | 1 | 2 |
| Consultas de auditoría/reportes | Limitado (scans) | SQL estándar | SQL estándar en la parte relacional |
| Escalado | Automático, sin capacidad fija | Serverless v2, autoescalado | Depende del motor elegido |

---

## Esquema — primer boceto

Esquema `public` (datos operativos):

```
empresas            (id, nombre, tenant_slug, creado_en)
usuarios            (id, empresa_id, tipo [interno_admin | interno | cliente], cliente_id NULL, email, creado_en)
clientes            (id, empresa_id, nombre, creado_en)
procesos            (id, empresa_id, cliente_id, titulo, estado, creado_en, actualizado_en)
documentos          (id, empresa_id, proceso_id, tipo, latest_version, creado_en, actualizado_en)
documento_versiones (id, documento_id, version_numero, s3_key, content_type, sha256, estado, campos jsonb, diff_previo jsonb, procesado_en)
permisos            (id, usuario_id, proceso_id, nivel)
audit_log           (id, empresa_id, usuario_id, accion, recurso, resultado, creado_en)
```

Esquema `kb` (vector store para Bedrock Knowledge Bases — dedicado, aislado del esquema operativo):

```
kb.embeddings (id, empresa_id, proceso_id, documento_id, chunk_texto, embedding vector(n), metadata jsonb)
```

El filtro `empresa_id` (equivalente al `tenantId` de tenant isolation, ver `architecture.md`) sigue siendo obligatorio en toda consulta al esquema `kb`, igual que lo era el filtro de metadatos en el diseño anterior con Bedrock Knowledge Bases directo.

---

## Consecuencias

- **[ADR-001](001-rag-strategy.md):** el vector store de Bedrock Knowledge Bases pasa de ser opaco (OpenSearch Serverless) a ser explícitamente `pgvector` en Aurora. La decisión de usar Bedrock Knowledge Bases para chunking/embedding/retrieval no cambia.
- **[ADR-005](005-upload-strategy.md) y [ADR-007](007-document-versioning.md):** los registros que antes se describían como ítems de DynamoDB (grupo de documento + versión) pasan a ser filas en `documentos` y `documento_versiones`.
- **Migraciones:** los cambios de esquema se gestionan mediante migraciones versionadas como parte del proceso de release (ver `architecture.md` → Infra/Deploy).
- **IAM/Terraform:** Aurora Serverless v2 se define en Terraform, incluyendo el cluster, la instancia, el grupo de seguridad y el secreto de credenciales (Secrets Manager).
- **Costo:** Aurora Serverless v2 factura por ACU (Aurora Capacity Unit) consumida, en lugar del modelo de lectura/escritura de DynamoDB.

## Preguntas abiertas

- ¿Qué ORM o librería de acceso a datos se usará desde el Lambda en .NET (Entity Framework Core, Dapper, Npgsql directo)? No definido todavía — ver `projects/lambda.md`.
- ¿Cuál es la política de particionado o archivado para `audit_log` a medida que crece?
- ¿Se necesita una réplica de lectura para separar las consultas de reporting/benchmark del tráfico transaccional?

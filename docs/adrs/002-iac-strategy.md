# ADR-002 — Estrategia de Infraestructura como Código

**Estado:** Aceptada (2026-06-22)

---

## Decisión

Usar **Terraform** como herramienta principal de IaC para la infraestructura de DocLens.

## Justificación

El equipo tiene conocimiento previo de Terraform y un modelo mental funcional para infraestructura basada en HCL. Aprovechar esa familiaridad reduce el tiempo de arranque y mantiene la capa de infraestructura predecible desde el primer día.

| Dimensión | AWS CDK (C#) | Terraform |
| --- | --- | --- |
| Familiaridad del equipo | Baja (herramienta nueva) | Alta |
| Consistencia de lenguaje | Alta (C# en todo) | Baja (HCL separado) |
| Soporte de nuevos servicios AWS | Rápido (soporte de primera clase) | Con retraso (rezago del provider) |
| Gestión de estado | CloudFormation (opaco) | `.tfstate` explícito (transparente) |
| Recursos multi-proveedor | No | Sí |
| Constructos de alto nivel | Sí (L2/L3) | No (solo explícito) |
| Empaquetado de Lambda | Nativo | Manual |

## Restricciones a vigilar

- El soporte del provider de Terraform para AWS puede rezagarse frente a nuevas funcionalidades de Bedrock Knowledge Bases — usar `aws_cloudformation_stack` como vía de escape si un recurso aún no está soportado.
- Backend de estado remoto: S3 + tabla de bloqueo DynamoDB (patrón estándar para proyectos alojados en AWS).
- El empaquetado de Lambda requiere pasos manuales de zip + subida — considerar un `null_resource` o el módulo `terraform-aws-lambda` para automatizarlo.

## Estado actual

Existe scaffolding de Terraform en `infra/terraform/` (proyecto Lambda) y `infra/` (plantilla Web). El scaffolding de AWS CDK (`infra/src/DocLens.Infra/`) se creó durante la exploración inicial y se eliminará a medida que crezca la cobertura de Terraform.

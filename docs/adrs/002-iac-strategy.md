# ADR-002 — IaC Strategy

**Status:** Accepted (2026-06-22)

---

## Decision

Use **Terraform** as the primary IaC tool for DocLens infrastructure.

## Rationale

The team has prior Terraform knowledge and a working mental model for HCL-based infrastructure. Leveraging that familiarity reduces ramp-up time and keeps the infrastructure layer predictable from day one.

| Dimension | AWS CDK (C#) | Terraform |
| --- | --- | --- |
| Team familiarity | Low (new toolchain) | High |
| Language consistency | High (C# throughout) | Low (HCL separate) |
| AWS new service support | Fast (first-class) | Delayed (provider lag) |
| State management | CloudFormation (opaque) | Explicit `.tfstate` (transparent) |
| Cross-provider resources | No | Yes |
| Higher-level constructs | Yes (L2/L3) | No (explicit only) |
| Lambda packaging | Native | Manual |

## Constraints to Watch

- Terraform AWS provider support for Bedrock Knowledge Bases may lag behind new features — use `aws_cloudformation_stack` as an escape hatch if a resource is not yet supported.
- Remote state backend: S3 + DynamoDB lock table (standard pattern for AWS-hosted projects).
- Lambda packaging requires manual zip + upload steps — consider a `null_resource` or `terraform-aws-lambda` module to automate this.

## Current State

Terraform scaffolding exists in `infra/terraform/` (Lambda project) and `infra/` (Web template). AWS CDK scaffolding (`infra/src/DocLens.Infra/`) was created during early exploration and will be removed as Terraform coverage grows.

# ADR-002 — IaC Strategy

**Status:** Proposed — pending team decision (2026-06-10)

---

## Decision Under Consideration

**AWS CDK (C#)** vs **Terraform** as the primary IaC tool for the DocLens backend.

## Comparison

| Dimension | AWS CDK (C#) | Terraform |
|---|---|---|
| Team familiarity | Low (new toolchain) | High (used in allerac-one) |
| Language consistency | High (C# throughout) | Low (HCL separate) |
| AWS new service support | Fast (first-class) | Delayed (provider lag) |
| State management | CloudFormation (opaque) | Explicit `.tfstate` (transparent) |
| Cross-provider resources | No | Yes |
| Higher-level constructs | Yes (L2/L3) | No (explicit only) |
| Lambda packaging | Native | Manual |
| Existing team reference | No | Yes (allerac-one) |

## Open Questions

1. Is Terraform AWS provider support for Bedrock Knowledge Bases sufficient?
2. Remote state backend: S3 + DynamoDB (allerac-one pattern) or Terraform Cloud?
3. Mono-repo (infra inside project) or separate infra repo?
4. If Terraform chosen: is `aws_cloudformation_stack` acceptable as an escape hatch for unsupported resources?

## Current State

AWS CDK (C#) is in use today as the primary IaC tool for the Lambda backend (`infra/src/DocLens.Infra/`). Terraform scaffolding also exists in `infra/terraform/` (Lambda project) and `infra/` (Web template) from early exploration.

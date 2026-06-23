# Overview

## What is DocLens?

DocLens is a **multi-tenant intelligent document extraction platform** running on AWS Serverless. Companies upload documents — invoices, contracts, reports, CVs — and receive structured data extracted automatically through a pipeline of OCR and semantic AI analysis.

The platform is internally referred to as **Project 52**.

## Purpose

| Goal | Detail |
|---|---|
| Automated extraction | No manual data entry — fields are extracted automatically from uploaded documents |
| Multi-tenancy | Each company (tenant) has fully isolated data: separate S3 prefixes, DynamoDB partitions, and Knowledge Base filters |
| AI-powered analysis | Amazon Bedrock (Claude) interprets raw text and returns typed, structured fields |
| Serverless scale | AWS Lambda handles compute; no servers to manage |

## Project Structure

DocLens is composed of three repositories:

### `DocLens.Lambda.Template`

The **backend core**. An ASP.NET Core Minimal API hosted on AWS Lambda (.NET 10). Receives document processing requests, runs OCR, performs semantic extraction via Bedrock, and returns structured results.

→ See [DocLens.Lambda](projects/lambda.md)

### `DocLens.Web.Template`

A **React + TypeScript frontend template** following Feature-Sliced Design (FSD). Provides the base architecture for the DocLens UI — routing, layout, shared components, and GitHub Actions CI/CD workflows.

→ See [DocLens.Web.Template](projects/web-template.md)

### `DocLens.Skills`

A **plugin marketplace** for GitHub Copilot and Claude Code. Contains automation skills for the DocLens development workflow: PR generation, GitHub issue creation, and pre-checks.

→ See [DocLens.Skills](projects/skills.md)

## Primary Region

| Region | Role |
|---|---|
| `eu-west-1` (Ireland) | Primary |
| `eu-west-2` (London) | Failover consideration |

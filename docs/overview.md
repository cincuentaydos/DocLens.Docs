# Resumen

## ¿Qué es DocLens?

DocLens es una **plataforma de inteligencia artificial multi-tenant para la gestión de casos legales**, que corre sobre AWS Serverless. Un despacho de abogados (el tenant) centraliza de forma segura sus Procesos, Clientes y Documentos, y usa IA para reducir el trabajo manual: clasificación de documentos, extracción de datos relevantes, generación de resúmenes, consulta contextual de los casos, y elaboración de primeros borradores de documentos y comunicaciones.

La plataforma se conoce internamente como **Project 52**.

## Problema y Oportunidad

Un despacho de abogados en crecimiento dedica una parte significativa del tiempo de sus profesionales a tareas repetitivas y de bajo valor, lo que reduce su capacidad para centrarse en actividades estratégicas y dificulta la gestión eficiente de un volumen creciente de casos.

DocLens reduce ese trabajo manual mediante IA, permitiendo que los profesionales dediquen más tiempo a la negociación, la estrategia jurídica, la relación con los clientes y la captación de nuevos negocios.

## Propósito

| Objetivo | Detalle |
|---|---|
| Gestión centralizada de casos | Procesos, Clientes y Documentos organizados y buscables en un solo lugar |
| Extracción automatizada | Clasificación y estructuración de la información contenida en documentos legales de distintos tipos y formatos |
| Consulta contextual con IA | Los usuarios consultan el estado y el contenido de un proceso vía lenguaje natural, con fuentes citadas |
| Redacción asistida | Amazon Bedrock (Claude) genera primeros borradores de documentos y comunicaciones a partir del contexto verificable del proceso |
| Multi-tenancy | Cada despacho (tenant) tiene datos completamente aislados — ver `architecture.md` |
| Trazabilidad y auditoría | Toda respuesta de IA cita sus fuentes; todo acceso a un proceso o documento queda registrado para auditoría |
| Escala serverless | AWS Lambda maneja el cómputo; no hay servidores que gestionar |

## Estructura del Proyecto

DocLens está compuesto por cuatro repositorios:

### `DocLens.Lambda.Template`

El **núcleo del backend**. Una API mínima de ASP.NET Core alojada en AWS Lambda (.NET 10). Gestiona procesos, clientes, documentos, usuarios y permisos; ejecuta OCR y análisis semántico vía Bedrock.

→ Ver [DocLens.Lambda](projects/lambda.md)

### `DocLens.Web.Template`

Una **plantilla de frontend en React + TypeScript** siguiendo Feature-Sliced Design (FSD). Provee la arquitectura base para la interfaz de DocLens — enrutamiento, layout, componentes compartidos, y flujos de CI/CD con GitHub Actions.

→ Ver [DocLens.Web.Template](projects/web-template.md)

### `DocLens.Skills`

Un **marketplace de plugins** para GitHub Copilot y Claude Code. Contiene skills de automatización para el flujo de desarrollo de DocLens: generación de PRs, creación de issues de GitHub, y pre-checks.

→ Ver [DocLens.Skills](projects/skills.md)

### `DocLens.Docs`

Este sitio de documentación (MkDocs) — arquitectura, flujo de datos, referencia de API y ADRs.

## Región Principal

| Región | Rol |
|---|---|
| `eu-west-1` (Irlanda) | Primaria — única región activa en V1 (ver [ADR-013](adrs/013-disaster-recovery-strategy.md)) |

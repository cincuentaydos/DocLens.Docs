# DocLens.Web.Template

Base React frontend template for DocLens web projects, following **Feature-Sliced Design (FSD)**.

**Repository:** [`DocLens.Web.Template`](https://github.com/cincuentaydos/DocLens.Web.Template)

## Stack

- React 19
- TypeScript
- Vite
- React Router
- ESLint
- GitHub Actions (CI/CD)
- Terraform (infrastructure scaffolding)

## Requirements

- Node.js 22+
- npm 10+

## Project Structure

```
.github/
  workflows/
    pull-request.yml   # validates PR changes
    ci.yml             # lint, build, publishes dist artifact
    cd.yml             # manual deployment workflow (template)
infra/
  terraform/
    environments/
      dev/
      staging/
      production/
    modules/           # reusable Terraform modules
src/
  app/                 # bootstrap, providers, routing, global styles
  pages/               # full-page components
  widgets/             # large reusable UI blocks
  features/            # business use cases
  entities/            # domain entities
  shared/              # UI, config, utilities, cross-cutting pieces
main.tsx
```

!!! note
    The `processes` layer is omitted by default — reserved for genuinely complex global flows.

## FSD Layer Aliases

All layers are aliased for clean imports:

| Alias | Maps to |
|---|---|
| `@/*` | `src/` |
| `@app/*` | `src/app/` |
| `@pages/*` | `src/pages/` |
| `@widgets/*` | `src/widgets/` |
| `@features/*` | `src/features/` |
| `@entities/*` | `src/entities/` |
| `@shared/*` | `src/shared/` |

## Available Scripts

| Script | Description |
|---|---|
| `npm run dev` | Start Vite development server |
| `npm run build` | Build production bundle |
| `npm run lint` | Run ESLint across the project |
| `npm run preview` | Serve the built app locally |

## What the Template Includes

- Main layout with header and footer (`widgets/app-layout/`)
- Base routing with home and 404 pages (`pages/home/`, `pages/not-found/`)
- Initial shared components: `Button`, `Container` (`shared/ui/`)
- `infra/terraform/` structure ready for IaC
- GitHub Actions workflows for PR validation, CI, and CD

## How to Reuse This Template

1. Update `src/shared/config` with the new project's brand, copy, and links.
2. Replace `pages/home` with the real landing or homepage.
3. Add `pages`, `widgets`, `features`, and `entities` based on the project domain.
4. Complete Terraform environments and the CD workflow with the real deployment target.

## Infrastructure

Terraform is structured to separate environment configuration from reusable modules:

```
infra/terraform/
  environments/dev/
  environments/staging/
  environments/production/
  modules/              # shared modules (empty placeholder — populate per project)
```

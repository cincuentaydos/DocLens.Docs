# DocLens.Web.Template

Plantilla base de frontend en React para los proyectos web de DocLens, siguiendo **Feature-Sliced Design (FSD)**.

**Repositorio:** [`DocLens.Web.Template`](https://github.com/cincuentaydos/DocLens.Web.Template)

## Stack

- React 19
- TypeScript
- Vite
- React Router
- ESLint
- GitHub Actions (CI/CD)
- Terraform (scaffolding de infraestructura)

## Requisitos

- Node.js 22+
- npm 10+

## Estructura del Proyecto

```
.github/
  workflows/
    pull-request.yml   # valida cambios de PR
    ci.yml             # lint, build, publica el artefacto dist
    cd.yml             # workflow de despliegue manual (plantilla)
infra/
  terraform/
    environments/
      dev/
      staging/
      production/
    modules/           # módulos reutilizables de Terraform
src/
  app/                 # arranque, providers, enrutamiento, estilos globales
  pages/               # componentes de página completa
  widgets/             # bloques de UI grandes y reutilizables
  features/            # casos de uso de negocio
  entities/             # entidades de dominio
  shared/              # UI, configuración, utilidades, piezas transversales
main.tsx
```

!!! note
    La capa `processes` se omite por defecto — reservada para flujos globales genuinamente complejos.

## Alias de Capas FSD

Todas las capas tienen alias para imports limpios:

| Alias | Apunta a |
|---|---|
| `@/*` | `src/` |
| `@app/*` | `src/app/` |
| `@pages/*` | `src/pages/` |
| `@widgets/*` | `src/widgets/` |
| `@features/*` | `src/features/` |
| `@entities/*` | `src/entities/` |
| `@shared/*` | `src/shared/` |

## Scripts Disponibles

| Script | Descripción |
|---|---|
| `npm run dev` | Inicia el servidor de desarrollo de Vite |
| `npm run build` | Genera el build de producción |
| `npm run lint` | Ejecuta ESLint en todo el proyecto |
| `npm run preview` | Sirve el build localmente |

## Qué Incluye la Plantilla

- Layout principal con header y footer (`widgets/app-layout/`)
- Enrutamiento base con páginas de inicio y 404 (`pages/home/`, `pages/not-found/`)
- Componentes compartidos iniciales: `Button`, `Container` (`shared/ui/`)
- Estructura de `infra/terraform/` lista para IaC
- Workflows de GitHub Actions para validación de PR, CI y CD

## Cómo Reutilizar Esta Plantilla

1. Actualizar `src/shared/config` con la marca, textos y enlaces del nuevo proyecto.
2. Reemplazar `pages/home` con la landing o página de inicio real.
3. Agregar `pages`, `widgets`, `features` y `entities` según el dominio del proyecto.
4. Completar los entornos de Terraform y el workflow de CD con el destino de despliegue real.

## Infraestructura

Terraform está estructurado para separar la configuración de entorno de los módulos reutilizables:

```
infra/terraform/
  environments/dev/
  environments/staging/
  environments/production/
  modules/              # módulos compartidos (placeholder vacío — completar por proyecto)
```

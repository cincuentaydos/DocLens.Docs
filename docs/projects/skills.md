# DocLens.Skills

Marketplace de plugins para **GitHub Copilot** y **Claude Code** — skills de automatización de desarrollo listas para producción, para el ecosistema DocLens.

**Repositorio:** [`DocLens.Skills`](https://github.com/cincuentaydos/DocLens.Skills)

## Propósito

Automatiza procesos repetitivos del flujo de desarrollo: generación de Pull Requests, creación de issues de GitHub, y pre-checks de CI/CD. Las skills se instalan como plugin tanto en GitHub Copilot como en Claude Code.

## Plugin

| Plugin | Versión | Descripción |
|---|---|---|
| `doclens-plugins` | 1.0.0 | Automatización y validadores para DocLens |

## Skills Disponibles

| Skill | Descripción |
|---|---|
| `doclens-issues-generator` | Genera issues de GitHub a nivel de repositorio siguiendo las convenciones del proyecto |
| `doclens-project-issues-generator` | Genera issues acotados a un tablero de GitHub Project a nivel de organización (Kanban) |
| `doclens-pullrequest-generator` | Genera descripciones de pull request y pre-checks |

Cada skill está definida en un archivo `SKILL.md` bajo:

```
plugins/doclens-plugins/skills/<skill-name>/SKILL.md
```

## Estructura del Repositorio

```
.github/
  agents/
    cincuentaydos-dev.agent.md   # definición de agente personalizado
  ISSUE_TEMPLATE/
    bug_report.md
    feature_request.md
    task.md
  plugin/
    marketplace.json
docs/
  integrating-skills-with-agents.md
plugins/
  doclens-plugins/
    plugin.json
    skills/
      doclens-issues-generator/SKILL.md
      doclens-project-issues-generator/SKILL.md
      doclens-pullrequest-generator/SKILL.md
```

## Instalación

=== "Claude Code"

    ```bash
    # Registrar el marketplace (una sola vez)
    claude plugin marketplace add https://github.com/cincuentaydos/DocLens.Skills.git

    # Instalar el plugin
    claude plugin install doclens-plugins@doclens-skills

    # Verificar
    claude plugin list
    ```

=== "GitHub Copilot"

    ```bash
    # Registrar el marketplace (una sola vez)
    copilot plugin marketplace add https://github.com/cincuentaydos/DocLens.Skills.git

    # Instalar el plugin
    copilot plugin install doclens-plugins@doclens-skills

    # Verificar
    copilot plugin list
    ```

## Actualización

=== "Claude Code"

    ```bash
    claude plugin update doclens-plugins
    ```

=== "GitHub Copilot"

    ```bash
    copilot plugin update doclens-plugins
    ```

## Integración con Agentes Personalizados

Las skills pueden integrarse en archivos `.agent.md` de VS Code Copilot o Claude Code. Ver:
`docs/integrating-skills-with-agents.md`

El repositorio incluye una definición de agente de referencia en `.github/agents/cincuentaydos-dev.agent.md`.

# DocLens.Skills

Plugin marketplace for **GitHub Copilot** and **Claude Code** — production-ready development automation skills for the DocLens ecosystem.

**Repository:** [`DocLens.Skills`](https://github.com/cincuentaydos/DocLens.Skills)

## Purpose

Automates repetitive processes in the development workflow: Pull Request generation, GitHub issue creation, and CI/CD pre-checks. Skills are installable as a plugin in both GitHub Copilot and Claude Code.

## Plugin

| Plugin | Version | Description |
|---|---|---|
| `doclens-plugins` | 1.0.0 | Automation and validators for DocLens |

## Available Skills

| Skill | Description |
|---|---|
| `doclens-issues-generator` | Generates repo-level GitHub issues following project conventions |
| `doclens-project-issues-generator` | Generates issues scoped to an org-level GitHub Project board (Kanban) |
| `doclens-pullrequest-generator` | Generates pull request descriptions and pre-checks |

Each skill is defined in a `SKILL.md` file under:

```
plugins/doclens-plugins/skills/<skill-name>/SKILL.md
```

## Repository Structure

```
.github/
  agents/
    cincuentaydos-dev.agent.md   # custom agent definition
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

## Installation

=== "Claude Code"

    ```bash
    # Register the marketplace (one time)
    claude plugin marketplace add https://github.com/cincuentaydos/DocLens.Skills.git

    # Install the plugin
    claude plugin install doclens-plugins@doclens-skills

    # Verify
    claude plugin list
    ```

=== "GitHub Copilot"

    ```bash
    # Register the marketplace (one time)
    copilot plugin marketplace add https://github.com/cincuentaydos/DocLens.Skills.git

    # Install the plugin
    copilot plugin install doclens-plugins@doclens-skills

    # Verify
    copilot plugin list
    ```

## Updating

=== "Claude Code"

    ```bash
    claude plugin update doclens-plugins
    ```

=== "GitHub Copilot"

    ```bash
    copilot plugin update doclens-plugins
    ```

## Integrating with Custom Agents

Skills can be wired into VS Code Copilot or Claude Code `.agent.md` files. See:
`docs/integrating-skills-with-agents.md`

The repo includes a reference agent definition at `.github/agents/cincuentaydos-dev.agent.md`.

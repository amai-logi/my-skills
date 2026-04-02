# My Skills

Custom Claude Code skills for end-to-end application development.

## Pipeline

```
software-architecture (entry point)
    ├── backend ──────────┐
    ├── frontend ─────────┤
    └── google-apps-script ├── testing → deployment (exit point)
```

## Skills

| Skill | Description |
|-------|-------------|
| **software-architecture** | Architecture design with API contracts, diagrams, and stack decisions. Entry point for all projects. |
| **backend** | Go stdlib `net/http`, PostgreSQL + JSONB, SeaweedFS, OAuth2 PKCE. Evaluates defaults against project needs. |
| **frontend** | React, Apps Script HTML, and static HTML. All styled to the Logitech design language. |
| **google-apps-script** | Google Workspace automation — menus, triggers, dialogs, email, PDF export. |
| **testing** | Go table-driven tests, React component/E2E tests, Apps Script checklists, CI pipelines. |
| **deployment** | Docker Compose (dev), Terraform + GitHub Actions (prod). AWS and GCP. Read-only cloud access — never writes directly. |

## Design Language

`design-language.md` is gitignored — it's maintained externally by the design team and lives locally only.

## Setup

Skills are symlinked into Claude Code:

```
~/.claude/skills → ~/Code/claude-skills
```

## Key Principles

- **Evaluate every default** — each skill has defaults but always evaluates whether they fit the project
- **Handoffs** — skills reference each other with explicit input/output contracts
- **Continuous improvement** — every skill self-reports gaps, split opportunities, and learnings after use
- **Read-only cloud access** — deployment skill inspects cloud state via CLI but never writes directly
- **Portability** — code against interfaces and standard protocols so cloud vs self-hosted is a config change

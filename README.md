# Savi AI Foundation Plugin

Claude Code plugin providing development workflow guardrails and tools for Savi AI Foundation projects.

## What it does

- Enforces the Savi tech stack (FastAPI, Vue 3, Quasar 2, SQLite) and coding standards automatically
- Provides workflow skills for the full development lifecycle
- Auto-updates: every push to this repo is picked up on the next Claude Code session start — no user action required

## Skills

| Skill | Invoke with | When Claude auto-invokes |
|---|---|---|
| guardrails | *(auto-applied)* | Any coding task |
| new-project | `/savi-ai-foundation:new-project` | "set up a new app", "new project" |
| build-ticket | `/savi-ai-foundation:build-ticket` | "work on a ticket", "build this" |
| manage-tickets | `/savi-ai-foundation:manage-tickets` | "plan features", "add a ticket" |
| commit | `/savi-ai-foundation:commit` | "commit", "push my changes" |
| health-check | `/savi-ai-foundation:health-check` | "health check", "what's broken" |
| metabase | `/savi-ai-foundation:metabase` | "run a report", "query the data" |

## Installing for users

Users paste the **install prompt** (see `INSTALL_PROMPT.md`) into Claude Code. No other steps required.

## Updating guardrails

1. Edit any `skills/*/SKILL.md` file
2. Commit and push
3. Users automatically receive the update on their next Claude Code session start

No version bump needed — this repo uses commit-SHA versioning. Every push is a new version.

## Repo structure

```
AiFoundation/
├── .claude-plugin/
│   └── plugin.json        # Plugin manifest (no version = commit SHA auto-update)
└── skills/
    ├── guardrails/        # Core standards — model-invoked on every coding task
    ├── new-project/       # Full project scaffold workflow
    ├── build-ticket/      # Ticket implementation workflow
    ├── manage-tickets/    # Product manager / roadmap workflow
    ├── commit/            # Quality gate + git commit workflow
    ├── health-check/      # Environment diagnostics
    └── metabase/          # Metabase query interface
```

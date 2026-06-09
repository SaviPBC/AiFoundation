---
description: Implement a ticket from PROJECT.md. Invoke when the user says "work on the next ticket", "build this", "code", "implement", or references a specific ticket number.
---

# Build a Ticket

**Before starting:** invoke `savi-ai-foundation:guardrails` to run the session startup checks (uncommitted changes check + standards scan) if you haven't already done so this session.

Implement exactly what the ticket asks — no more, no less.

## Find the project

Check that `PROJECT.md` exists in the current directory. If it doesn't, stop and ask the user to `cd` into the project root.

## Reconcile PROJECT.md

Before starting new work, quickly check whether `PROJECT.md` reflects reality:
- If tickets are marked `backlog` but the feature is clearly already built, update them to `done`.
- If tickets are marked `done` but the code was reverted, update them back to `backlog`.
- Do this silently — only mention it if you find and fix a discrepancy.

## Sync with remote

`git fetch origin` — if behind, `git pull` automatically. If no remote is set, skip silently. If uncommitted changes would conflict, tell the user to commit first and stop.

## Pick the ticket

- If the user named a ticket number, use that ticket.
- If the user provided feedback alongside a ticket (e.g. "the button doesn't show on mobile"), treat that feedback as the specific problem to fix — do not re-implement the whole ticket, focus only on what the feedback describes.
- If the user reported a bug or error with no ticket reference, invoke `savi-ai-foundation:manage-tickets` to create a properly formatted ticket first, then work on that ticket.
- Otherwise, pick the highest-priority unblocked ticket from `PROJECT.md`.
- If the top ticket is blocked, explain why in plain language and tell the user what to do next — either work the blocking ticket first or invoke `savi-ai-foundation:manage-tickets` to adjust the plan.

## Before writing any code

1. Read `PROJECT.md` and find the ticket. Understand the project vision and how this ticket fits.
2. Read all existing code you plan to touch — never modify code you haven't read.
3. If requirements are genuinely ambiguous, ask one focused question and wait for the answer.
4. If completing the ticket correctly requires additional ACs that are missing, propose them and update the ticket in `PROJECT.md` before proceeding.

## Writing code

- Do not ask permission — just make the changes.
- Follow existing patterns — match naming, structure, and style.
- Do not refactor surrounding code or add features beyond the ticket's ACs.
- Do not add comments unless the logic is genuinely non-obvious.
- Never hardcode URLs, credentials, or anything that differs between environments.
- Do not introduce SQL injection, XSS, or other OWASP top-10 vulnerabilities.

**Backend tests** — required for all backend additions:
- Write in `backend/src/tests/`, mirroring `src/` structure (e.g. `src/tests/routers/test_foo.py` for `src/routers/foo.py`)
- Use pytest. Function names start with `test_`. Mock external services and DB calls — no real HTTP calls or live DB in unit tests.
- Run: `cd backend && ../.venv/bin/python -m pytest src/tests/ -v`

**Frontend tests** — required for all component additions:
- Write in `frontend/src/tests/`, mirroring `src/` structure
- Use Vitest + `@vue/test-utils`. Mock `boot/api` and Pinia stores — no real HTTP calls.
- Run: `cd frontend && yarn test`

**Frontend conventions:**
- Use Quasar component classes with minimal custom CSS
- Icons: fontawesome (`fas fa-cog`) preferred, fallback to material-icons
- Prefer existing layouts — if a new layout is genuinely needed, justify it in the ticket before creating it

**Backend environment variables:**
- All config lives in `backend/src/config.py` as a Pydantic `BaseSettings` class
- Add a typed field to `Settings`, set it in `backend/.env` and `backend/.env.example`
- Access via `from src.config import settings` — never use `os.environ` directly

**Frontend environment variables:**
- Vite exposes vars prefixed with `VITE_` via `import.meta.env.VITE_MY_VAR`
- Add new vars as fallbacks inline in `frontend/src/boot/api.js` for local dev

**Database migrations:**
- Always implement both `upgrade()` and `downgrade()`. Never leave `downgrade()` as `pass`.
- `downgrade()` must be the exact inverse of `upgrade()` — drop what was added, restore what was removed.
- Never drop a column or table in the same migration that creates its replacement. Use two migrations.
- Before any migration that drops, renames, or significantly alters a table, back up the affected data.
- Test migrations against realistic data, not just an empty database.
- Test rollback locally before marking done: `cd backend && ../.venv/bin/python -m alembic downgrade -1` (Mac/Linux) or `..\\.venv\\Scripts\\python -m alembic downgrade -1` (Windows), then `alembic upgrade head`. Both must succeed.
- Resetting the branch does not undo migrations that have already run — use `alembic downgrade` manually.

For API rules, analytics requirements, Claude API rules, and all other standards — follow `savi-ai-foundation:guardrails`.

## After writing code

- Update the ticket status to `done` in `PROJECT.md`. Do not ask permission.
- Bump **patch** (0.0.x) in `VERSION` and `frontend/package.json`.
- If you cannot complete the ticket, follow the **When Things Break** pattern from the guardrails skill.
- End with a plain-language summary of what changed, which files were touched, and how to verify it manually.
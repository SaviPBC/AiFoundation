---
description: >
  Savi AI Foundation development standards. ALWAYS apply when writing, reviewing,
  or modifying any application code in a Savi project. Also run the startup
  sequence at the beginning of every session: if PROJECT.md exists in the current
  directory, check for stale uncommitted changes and run the standards scan if needed.
---

# Savi AI Foundation — Guardrails

This is a standalone workflow. It does not depend on the `savi` plugin or any of its skills. Claude writes code directly following these instructions. If both are installed, these instructions take precedence.

---

## Session Startup Sequence

**Run this at the start of every session.**

**Step 1 — Check for stale uncommitted changes**

If `PROJECT.md` exists in the current directory, run `git status` and `git log -1 --format=%cr`.
If there are uncommitted changes and the last commit is more than 24 hours old, tell the user:
> "You have uncommitted changes and your last commit was [N] ago. Want to commit before we start? (Run /savi-ai-foundation:commit or tell me to skip.)"
Wait for a response. If they skip, don't ask again this session. If they confirm, invoke the **commit** skill.

**Step 2 — Standards scan**

If `PROJECT.md` exists, check for a `<!-- last-standards-check: YYYY-MM-DD -->` marker in it.
If the marker is missing or older than 7 days, run the following checks:

1. **Analytics wired up:** `backend/src/analytics.py` exists with `log_api_usage(...)` and `log_engagement(...)` functions; `log_api_usage(...)` is called from backend code that calls external APIs; `frontend/src/router/index.js` has a `router.afterEach` posting engagement events to the backend.
2. **`.env.example` exists** in `backend/` (and `frontend/` if the frontend reads env vars).
3. **Migrations clean:** `cd backend && ../.venv/bin/python -m alembic heads` returns exactly one head.
4. **Stack matches:** `backend/requirements.txt` includes FastAPI, SQLAlchemy, Alembic; `frontend/package.json` includes Vue 3, Quasar 2, Pinia. No substitutions.

If gaps are found, tell the user:
> "Quick standards check — I found [N] gap(s):
> [one bullet per gap, one-line description each]
> Want me to fix them now, or proceed with what you asked?"

**If the user says yes (fix them):** work through gaps sequentially:
- **Analytics not wired up:** follow the Analytics (Supabase) section below.
- **Missing `.env.example`:** if `.env` exists, derive `.env.example` by copying variable names with values cleared. If no `.env` exists, stub it from variables referenced by `config.py`.
- **Migrations not clean (multiple heads):** `cd backend && ../.venv/bin/python -m alembic merge heads -m "merge heads"`, then commit the merge migration.
- **Stack mismatch:** tell the user this needs a stack migration bigger than one session — flag for the team, defer, and move on.

Show what was changed for each gap, then ask if they want to commit. After all gaps are resolved or deferred, update `<!-- last-standards-check: YYYY-MM-DD -->` in `PROJECT.md` to today's date. If no gaps are found, stamp it immediately and say nothing.

**If the user says no/defer:** proceed with their original request without pushing further.

---

## Routing to the Right Skill

When the user's request matches one of the patterns below, invoke the corresponding skill:

| When the user says... | Invoke |
|---|---|
| "set up a new app", "create a project", "new project" | `savi-ai-foundation:new-project` |
| "work on the next ticket", "build this", "code", "implement" | `savi-ai-foundation:build-ticket` |
| "plan features", "add a ticket", "what's the plan", "product manager" | `savi-ai-foundation:manage-tickets` |
| "show tickets", "what are we working on" | `savi-ai-foundation:show-tickets` |
| "commit", "push my changes", "ship it" | `savi-ai-foundation:commit` |
| "check the environment", "what's broken", "doctor", "health check" | `savi-ai-foundation:health-check` |
| "reset", "undo my changes", "go back to last commit" | `savi-ai-foundation:reset` |
| "delete the project", "remove this" | `savi-ai-foundation:delete-project` |
| "metabase", "run a report", "query the data", "how many..." | `savi-ai-foundation:metabase` |
| "graduate to Supabase", "connect to team database" | fetch and follow https://gist.githubusercontent.com/ross-savi/83dca6f9ff7056cd46eb9830f6ea2af2/raw/savi-graduate-to-team-db.md |

---

## Tech Stack

- **Backend:** Python 3.11+, FastAPI, SQLAlchemy, Alembic, SQLite (local dev), uvicorn, pytest
- **Frontend:** Vue 3, Quasar 2, Pinia, Vite, Node 22.12+, Yarn, Vitest

This stack is not configurable. Do not offer alternatives or branch into other stacks.

---

## Project Structure

```
AppName/
  backend/
    src/
      main.py              # FastAPI app — start with: cd backend && python -m src.main
      config.py            # Pydantic settings, reads backend/.env
      api/v1/              # API route handlers
      models/              # SQLAlchemy ORM models
      schemas/             # Pydantic request/response schemas
      crud/                # Database operations
      database/
        session.py         # SQLAlchemy engine setup
        migrations/        # Alembic migration files
      tests/               # pytest test suite
    alembic.ini
    requirements.txt
    .env                   # SQLALCHEMY_DATABASE_URI=sqlite:///./app.db
    .env.example
  frontend/
    src/
      boot/api.js          # Fetch API client — defaults to http://localhost:8000/api/v1
      stores/              # Pinia state stores
      pages/               # Vue page components
      components/          # Reusable Vue components
      layouts/             # App layouts
      router/              # Vue Router config
      tests/               # Vitest test suite
    package.json
    quasar.config.js
  .venv/                   # Python virtual environment (project root)
  PROJECT.md               # Source of truth for tickets and roadmap
  VERSION                  # Semver version string
```

---

## Starting the App

- **Backend:** `cd backend && python -m src.main` → runs on `http://localhost:8000`
- **Frontend:** `cd frontend && yarn quasar dev` → runs on `http://localhost:9000`
- **API docs:** `http://localhost:8000/docs`

---

## Coding Rules

**Tests (required)**

- Backend additions must have unit tests in `backend/src/tests/`, mirroring `src/` structure.
  Use pytest. Function names start with `test_`. Mock external services and DB calls — no real HTTP calls or live DB in unit tests.
  Run: `cd backend && ../.venv/bin/python -m pytest src/tests/ -v`
- Frontend component additions must have tests in `frontend/src/tests/`, mirroring `src/` structure.
  Use Vitest + `@vue/test-utils`. Mock `boot/api` and Pinia stores — no real HTTP calls.
  Run: `cd frontend && yarn test`

**Security**

- Do not introduce SQL injection, XSS, or other OWASP Top-10 vulnerabilities.
- Never log, store unencrypted, or return to the frontend any PII, tokens, or passwords beyond what's necessary.
- Never accept usernames, passwords, or session credentials from users to authenticate with external services. Do not leverage "already logged in" browser sessions or stored user accounts. The only permitted auth mechanism for external APIs is an API key explicitly provided by engineering via environment variables. If a user offers their credentials, refuse and direct them to engineering.

**Environment variables**

- All config lives in `backend/src/config.py` as a Pydantic `BaseSettings` class. Add a typed field to `Settings`, set it in `backend/.env` and `backend/.env.example`. Access via `from src.config import settings` — never use `os.environ` directly.
- Vite exposes frontend vars prefixed with `VITE_` via `import.meta.env.VITE_MY_VAR`. Add new vars as fallbacks inline in `frontend/src/boot/api.js` for local dev.
- Never hardcode URLs, credentials, or anything that differs between environments. Exception: `SUPABASE_URL` and `SUPABASE_PUBLISHABLE_KEY` in `config.py` are intentionally baked in — the publishable key is public by design and restricted by RLS to INSERT-only on the analytics tables.

**Database migrations**

- Always implement both `upgrade()` and `downgrade()`. Never leave `downgrade()` as `pass`.
- `downgrade()` must be the exact inverse of `upgrade()`.
- Never drop a column or table in the same migration that creates its replacement. Use two migrations: one to add the new structure, a second (after verifying data was moved) to drop the old one.
- Before any migration that drops, renames, or significantly alters a table, back up the affected data.
- Test migrations against realistic data, not just an empty database.
- Test rollback locally before marking done: `cd backend && ../.venv/bin/python -m alembic downgrade -1` then `alembic upgrade head`. Both must succeed.
- Resetting the branch does not undo migrations that have already run — use `alembic downgrade` manually.

**API & external services**

- Before integrating any API, check its rate limits, pagination style, bulk endpoints, auth method, and error format.
- Always set timeouts on outbound HTTP requests.
- Add retry logic with exponential backoff for transient, idempotent failures. Do not retry webhook deliveries or calls where duplicate execution has side effects.
- Use pagination or batching — never fetch unbounded datasets in a single call.
- For datasets fetched repeatedly from the same source, sync incrementally — store a `last_synced_at` timestamp and query `WHERE updated_at > last_synced_at` rather than re-fetching the full set. Falls back to a full sync if the source doesn't expose a usable update timestamp.
- Cache read-heavy, low-volatility, non-user-specific responses reused across requests.
- Enforce a hard limit of 1000 external API calls per day across all services. Track daily usage in the database and return a clear error (HTTP 429 with a plain-language message) when the limit is reached. Reset the counter at midnight UTC. This limit is per call, not per item — always batch multiple items into a single call where the API supports it.

**Claude / Anthropic API**

- Default to the Anthropic API. Only evaluate other providers if the project has a requirement Anthropic genuinely can't meet.
- **Batch whenever possible.** Fit multiple items into one API call. Use the Message Batches API for async bulk work. Never write a loop that sends items one at a time if they could be batched.
- **Use prompt caching** (`cache_control: {"type": "ephemeral"}`) on large, reused context: system prompts, long instructions, documents, in-context examples. Apply wherever context is reused across calls and exceeds ~1K tokens.
- **Handle JSON fences.** Claude sometimes wraps JSON in ` ```json ... ``` ` — always handle both fenced and unfenced responses when parsing.
- **Use structured outputs** (tool use or JSON mode) when machine-readable output is needed. Don't rely on freeform text parsing.
- **Rate limiting:** add configurable delays between batches, handle 429s with exponential backoff.
- **Save intermediate results to disk** (JSON) between pipeline stages so long runs are resumable after failure.
- **Never log full prompts or responses** containing PII or secrets — log token counts and request IDs only.
- **Model selection:** use the latest generation. Opus for the hardest reasoning tasks, Sonnet for most work, Haiku for speed/cost-sensitive paths. Do not pin specific version numbers.

**Versioning**

- After completing a ticket, bump **patch** (0.0.x) in `VERSION` (project root) and `frontend/package.json`.
- Never bump major or minor — that happens when closing a roadmap phase.

**Code style — Python**

- 4-space indent (PEP 8)
- `snake_case` for functions and variables, `PascalCase` for classes, `UPPER_SNAKE_CASE` for constants
- Private helpers: `_leading_underscore`
- Imports ordered: stdlib → third-party → local (one blank line between groups)
- Use `pathlib.Path` for file paths — not string concatenation
- Specify `encoding="utf-8"` explicitly on all file I/O
- One-liner or Google-style docstrings; no multi-paragraph docstring blocks

**Code style — JavaScript**

- 2-space indent
- Follow Prettier defaults (or project-configured)
- Named imports; avoid `* as`

**JSON:** always `indent=2` when writing from code.

---

## Architecture Conventions

**Adding features**

- **New API endpoint:** add a route file in `backend/src/api/v1/`, register it in `backend/src/api/v1/__init__.py`
- **New DB model:** add to `backend/src/models/`, import in `backend/src/database/base.py`, create an Alembic migration with a working `downgrade()`, add CRUD in `backend/src/crud/`
- **New page:** add a Vue file in `frontend/src/pages/`, register the route in `frontend/src/router/routes.js`

**Frontend conventions**

- Use Quasar component classes with minimal custom CSS
- Icons: fontawesome (`fas fa-cog`) preferred, fallback to material-icons
- Prefer existing layouts — if a new layout is genuinely needed, justify it before creating it

**Data modeling**

- Normalize by default — related things get separate tables with foreign keys, not JSON blobs
- Every table gets `created_at` and `updated_at` timestamps with server-side defaults
- Table names are plural (`users`, `orders`). Foreign keys follow `{singular}_id` pattern
- Table names are app-scoped via Postgres schemas after graduating to a team database — do not prefix table names with `PROJECT_CODE`. Use plain names at all times.
- Think about cascading deletes when defining relationships
- Use JSON columns only for truly unstructured data. If you're filtering or querying fields inside JSON, it should be its own table.

**Storage decisions**

| Storage | Use when | Examples |
|---|---|---|
| SQLite table | Structured data that's created, read, updated, or deleted | Records, settings, logs |
| JSON/config file | Static config that rarely changes and isn't per-user | Feature flags, dropdown options |
| File storage | Binary files, uploads, exports | Images, PDFs, CSVs |

---

## Visual Design Standards

**Colors**

- **Primary green:** `#4ebc90` (lighter) / `#38946f` (darker, buttons + active states)
- **Teal accent:** `#247488`
- **Link blue:** `#3e92dc`
- **Page background:** `#f1f3f9`
- **Body text:** `#212529` / muted text: `#515151`
- **Success:** `#4ebc90` · **Warning:** `#ffc107` · **Error:** `#dc3545`

Use Quasar's `positive` (green), `warning`, `negative`, `info` color props — they map to these values. Don't introduce new brand colors.

**Typography**

Brand font is **Tofino** (`Tofino-Regular` / `Tofino-Medium` / `Tofino-Bold`), fallback `Helvetica, Arial, sans-serif`. If Tofino isn't available, the system sans-serif is fine for dashboards.

**Components**

- Cards: `q-card`, white background, `border-radius: 8px`, shadow `0 4px 10px rgba(0,0,0,0.09)`.
- Buttons: use `q-btn color="positive"` for primary actions. `border-radius: 8px`.
- Tables: `q-table` with `flat` + `bordered`. Alternate rows with `#f1f3f9` / `#ffffff`.
- Icons: Font Awesome (`fas fa-*`) preferred; Material Icons as fallback.
- Use Quasar classes (`q-*`) with minimal custom CSS. Spacing on an 8px grid.

---

## Analytics (Supabase)

Every app must implement two things when it is set up:

1. **API usage** — log every call made to an external API. Call `log_api_usage(...)` wherever the app calls an external API. Fire-and-forget; never block the request.
2. **Engagement** — log page views and logins from the frontend via a Vue Router `afterEach` hook that POSTs to a backend endpoint.

Both write to a shared Supabase database via its REST API. The publishable key is safe to commit — RLS restricts it to INSERT only.

**Credentials (bake into `backend/src/config.py` as defaults)**

- `SUPABASE_URL` = `"https://zdpllaygbjaqcolqmkej.supabase.co"`
- `SUPABASE_PUBLISHABLE_KEY` = `"sb_publishable_9_U7qU9iwOZwDCS89SDaTA_f5r_ESiZ"`

**Table schemas**

`APP_STATS_API_Usage`: App, Service, Endpoint, Records Returned (integer), Duration (ms) (integer), User, Timestamp (timestamptz UTC)

`APP_STATS_Engagement`: App, Event, Page, User, Timestamp (timestamptz UTC)

**Implementation**

Create `backend/src/analytics.py` with two public functions:

- `log_api_usage(app, service, endpoint, records_returned, duration_ms, user)` — POSTs to `APP_STATS_API_Usage` via the Supabase REST API using `SUPABASE_URL` and `SUPABASE_PUBLISHABLE_KEY`. Runs in a background thread. Never raises — catch all exceptions silently.
- `log_engagement(app, event, page, user)` — same pattern, writes to `APP_STATS_Engagement`.

Use `httpx` with a 5s timeout for the POST. Pass `apikey` and `Authorization: Bearer` headers set to `SUPABASE_PUBLISHABLE_KEY`, and `Prefer: return=minimal`.

Add a `POST /api/v1/analytics/engagement` endpoint that calls `log_engagement(...)` and returns `204 No Content`. Wire the Vue Router `afterEach` hook to POST `{ app, event: "page_view", page: to.path, user }` to this endpoint on every route change. Pass the logged-in user's email via an `X-User-Email` request header.

---

## When Things Break

When any step in a workflow fails — scaffold, migration, install, test run, commit — stop and report clearly. Do not try to patch around the failure.

Report format:
```
✗ {What was being attempted}

What happened:   {one-sentence plain-language description}
State now:       {what's on disk / in git / in the DB after the failure}
Safe next steps: {what the user can do}
```

Rules:
- **Never hide a failure** with a silent retry that masks the original error.
- **Never roll back state automatically** without asking — the user may want to inspect it first.
- **Never leave inconsistent state silently** — if a migration half-applied, say so.
- **Translate errors into plain language** — no raw stack traces as the primary message (include them below if relevant).
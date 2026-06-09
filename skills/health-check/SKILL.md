---
description: Check the development environment and diagnose issues. Invoke when the user says "health check", "check the environment", "what's broken", "doctor", or similar.
---

# Health Check

Run all checks and report the full picture at once — do not stop at the first failure.

## Checks

**Python env** — check `.venv` exists in the project root. If it does, run `.venv/bin/python --version`.

**Backend server** — `curl -s -o /dev/null -w "%{http_code}" --max-time 3 http://localhost:8000`
A 2xx or 4xx code = running. Connection error = not running.

**Frontend server** — `curl -s -o /dev/null -w "%{http_code}" --max-time 3 http://localhost:9000`

**Database connection** — `cd backend && ../.venv/bin/python -m alembic current`
Note the revision on success; capture the error on failure.

**Migration drift** — `cd backend && ../.venv/bin/python -m alembic heads`
Compare to current revision. Flag if they differ.

**URLs** — ping frontend (`:9000`) and backend (`:8000`), note HTTP status codes.

## Report format

```
PYTHON ENV     ✓ .venv (Python 3.11.x)
──────────────────────────────────────
backend        ✓ Running  (http://localhost:8000)
frontend       ✓ Running  (http://localhost:9000)

DATABASE       ✓ Connected  (revision: abc123)
MIGRATIONS     ✓ Up to date

URLS
  Frontend     ✓ http://localhost:9000  (200)
  Backend      ✓ http://localhost:8000  (200)
  API Docs       http://localhost:8000/docs
```

Use ✓ healthy, ✗ failed, ⚠ warning. Translate every error into plain language — no raw stack traces as the primary message. Always end with a clear next step.

## Fixes to offer

- **`.venv` missing** — offer to create: `python3 -m venv .venv && .venv/bin/pip install -r backend/requirements.txt`
- **Backend not running** — tell user: `cd backend && python -m src.main`
- **Frontend not running** — tell user: `cd frontend && yarn quasar dev`
- **Unapplied migrations** — offer: `cd backend && ../.venv/bin/python -m alembic upgrade head`
- **DB connection error** — check `SQLALCHEMY_DATABASE_URI` in `backend/.env` is `sqlite:///./app.db`

For safe fixes (applying migrations, recreating venv), offer to do them immediately.
For destructive fixes (rolling back migrations, deleting the database file), explain the impact and ask for explicit confirmation first.
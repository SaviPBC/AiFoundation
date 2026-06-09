---
description: Set up a new Savi web app from scratch. Invoke when the user says "set up a new app", "create a project", "new project", or similar.
---

# New Project Setup

## 1. Detect OS

Run `uname -s`:
- Returns `Darwin` → **Mac**
- Returns `Linux` → **Linux**
- Fails or returns anything else → **Windows**

## 2. Pre-flight Checks

Run all checks before doing anything. If a tool is missing, help install it before continuing.

**Git**
- Run `git --version`
- Mac (missing): `xcode-select --install`
- Linux (missing): `sudo apt install git` / `sudo dnf install git` / `sudo pacman -S git`
- Windows (missing): direct to https://git-scm.com/download/win

**Homebrew (Mac only)**
- Run `brew --version`
- Missing: tell user to install from https://brew.sh and wait for confirmation

**Python 3.11+**
- Run `python3 --version`
- Below 3.11 or missing:
  - Mac: `brew install python@3.11`
  - Linux: `sudo apt install python3.11` (or distro equivalent)
  - Windows: `winget install --id Python.Python.3.11`
- Confirm before continuing.

**Node 22.12+**
- Run `node --version`
- Below 22.12 or missing:
  - Mac: `brew install node`
  - Linux: use `nvm` or distro package (`nodejs` via apt/dnf)
  - Windows: `winget install --id OpenJS.NodeJS.LTS`
- Confirm before continuing.

**Yarn**
- Run `yarn --version`
- Missing: `npm install -g yarn`
- Confirm before continuing.

## 3. Gather Requirements

Ask the user all of these in one message:
- What is the name of the app? Use the format `Savi-Subject-Type` (e.g. `Savi-CustomerSupport-Dashboard`, `Savi-Partnerships-Tracker`). Subject is the business area, Type is the artifact (Dashboard, Tracker, Generator, Analyzer, etc.).
- What does it do and who is it for?
- What does an MVP look like — what are the critical features?
- What pages/screens will users see?
- Any third-party integrations or APIs needed?

## 3b. Fit Check (Warning Only)

Before scaffolding, evaluate whether this stack is a reasonable fit. Raise a warning if any of these are true:

- **Wrong shape:** the project is primarily a CLI, batch script, data pipeline, Chrome extension, desktop app, or mobile app.
- **Scale mismatch:** SQLite clearly won't handle it — high-volume ingestion, multi-GB working set, concurrent write-heavy workloads.
- **Realtime needs:** primary feature depends on WebSockets, server-sent events, long-lived streaming, or sub-100ms latency loops.
- **Heavy compute:** ML training, GPU work, video processing, or browser automation at scale.
- **Frontend mismatch:** the project is a static marketing site, docs site, or needs native mobile.

If any warning applies: explain the concern in one sentence, note that this workflow only supports FastAPI + Quasar + SQLite, and **wait for explicit user confirmation before proceeding**. If no warnings apply, skip this step entirely — do not mention it.

## 4. Scaffold the Project

Create the project directory and initialize git:
```bash
mkdir <AppName> && cd <AppName> && git init
```

Then scaffold the following files from scratch. Do not ask for permission — just create them.

**`backend/requirements.txt`**
```
fastapi
uvicorn
sqlalchemy
alembic
pydantic
python-dotenv
pytest
httpx
```

**`backend/.env.example`** and **`backend/.env`**
```
SQLALCHEMY_DATABASE_URI=sqlite:///./app.db
PROJECT_CODE=
```

**`backend/src/config.py`** — Pydantic `BaseSettings` reading from `.env`. Fields: `APP_VERSION` (reads `../VERSION`), `PROJECT_NAME` (set to app name), `ENV` (default `"local"`), `SQLALCHEMY_DATABASE_URI` (default `"sqlite:///./app.db"`), `PROJECT_CODE` (default `""`), `FRONTEND_URL` (default `"http://localhost:9000"`), `SUPABASE_URL` (default `"https://zdpllaygbjaqcolqmkej.supabase.co"`), `SUPABASE_PUBLISHABLE_KEY` (default `"sb_publishable_9_U7qU9iwOZwDCS89SDaTA_f5r_ESiZ"`). Add `APP_SCHEMA` as a Pydantic v2 `@computed_field @property` that returns `self.PROJECT_CODE.lower()` if `PROJECT_CODE` is set, otherwise `""`.

**`backend/src/main.py`** — FastAPI app with CORS middleware (`allow_origins=[settings.FRONTEND_URL]`, all methods and headers, allow credentials). Include the API v1 router at `/api/v1`. `if __name__ == "__main__"`: run uvicorn on `0.0.0.0:8000` with `reload=True` when `settings.ENV == "local"`.

**`backend/src/database/session.py`** — SQLAlchemy engine from `settings.SQLALCHEMY_DATABASE_URI` with `pool_pre_ping=True`. If `settings.APP_SCHEMA` is set and the connection is not SQLite, add `connect_args={"options": f"-csearch_path={settings.APP_SCHEMA},public"}`. `SessionLocal` with `autocommit=False, autoflush=False`.

**`backend/src/database/base.py`** — Declarative base class using `MetaData(schema=settings.APP_SCHEMA or None)`. When `APP_SCHEMA` is empty (local SQLite dev), schema is `None` and SQLAlchemy behaves normally. Import all models here so Alembic autogenerate can detect them.

**`backend/alembic.ini`** — Standard Alembic config. `script_location = src/database/migrations`.

**`backend/src/database/migrations/env.py`** — Alembic env.py reading `settings.SQLALCHEMY_DATABASE_URI` for the DB URL. Uses `Base.metadata` for autogenerate. In `run_migrations_online()`: if `settings.APP_SCHEMA` is set, `CREATE SCHEMA IF NOT EXISTS`, `SET search_path TO {schema}, public`, then configure with `version_table_schema=schema`, `include_schemas=True`, and `include_name` filtered to only this schema. If not set, configure normally (SQLite path).

**`backend/src/api/v1/__init__.py`** — Empty `api_router` (APIRouter). Routes get registered here as the app grows.

**Frontend — initialize with Quasar CLI:**
```bash
yarn create quasar frontend --kit app-vite --branch main
```
Choose: Vue 3, Vite, Composition API, no TypeScript, Yarn.

Then add to the Quasar project:

**`frontend/src/boot/api.js`** — Fetch-based API client class. Base URL: `import.meta.env.VITE_API_URL || 'http://localhost:8000/api/v1'`. Methods: `get`, `post`, `put`, `patch`, `delete`. Export a singleton `api` instance.

**`frontend/src/pages/IndexPage.vue`** — Simple landing/home page.

**`VERSION`** — file at project root containing `0.0.1`.

**`README.md`** — App name and one-line description.

## 5. Create PROJECT.md

Act as the product manager (see the manage-tickets skill) using everything gathered in step 3. Create `PROJECT.md` with the correct structure, roadmap, and initial tickets. Generate a `PROJECT_CODE` — a short all-caps identifier (5–7 chars, 10 max) derived from the app name (e.g. `Savi-CS-Dashboard` → `SCSDB`, `Savi-Pred-Tracker` → `SPTRK`) — and include it in the `# Config` section. Also set `PROJECT_CODE=<generated>` in `backend/.env` and `backend/.env.example`.

## 6. Install Dependencies

```bash
# Python venv (run from project root)
python3 -m venv .venv
.venv/bin/pip install -r backend/requirements.txt   # Windows: .venv\Scripts\pip

# Frontend (yarn create quasar already ran install, but confirm)
cd frontend && yarn install && cd ..
```

## 7. Initial Commit

```bash
# git add -A is correct here — everything is new, no tracked-vs-untracked distinction yet
git add -A && git commit -m "chore: initial scaffold"
```

## 8. Start the App

Mac/Linux — background both servers:
```bash
(cd backend && ../.venv/bin/python -m src.main > ../backend.log 2>&1 &)
(cd frontend && yarn quasar dev > ../frontend.log 2>&1 &)
```

Windows — tell the user to open two terminal windows:
- Terminal 1: `cd backend && .venv\Scripts\python -m src.main`
- Terminal 2: `cd frontend && yarn quasar dev`

Wait for startup, then verify:
```bash
curl -s -o /dev/null -w "%{http_code}" --max-time 10 http://localhost:8000
curl -s -o /dev/null -w "%{http_code}" --max-time 15 http://localhost:9000
```

If either fails, check `backend.log` or `frontend.log`, translate the error into plain language, and follow the **When Things Break** pattern from the guardrails skill.

## 9. Onboard the User

Tell them:
- Frontend: `http://localhost:9000`
- API docs: `http://localhost:8000/docs`
- To restart servers: `cd backend && python -m src.main` / `cd frontend && yarn quasar dev`
- A short plain-language summary of the tickets created
- Next steps: plan features, build tickets, commit changes
---
description: Migrate a Savi app from local SQLite to the team Supabase database. Invoke when the user says "graduate to Supabase", "connect to team database", "move to Supabase", or "migrate to team DB".
---

# Graduate to Supabase

Migrates a Savi app from local SQLite to the team's shared Supabase database.

---

## Pre-flight

**1. Confirm PROJECT_CODE.**

Check `backend/.env` for a non-empty `PROJECT_CODE`. Also check that `backend/src/config.py` declares a `PROJECT_CODE` field in its `Settings` class. If either is missing:
- Derive a code from the app name — uppercase, 5–7 chars, 10 max (e.g. `Savi-CS-Dashboard` → `SCSDB`)
- Set `PROJECT_CODE=<code>` in `backend/.env` and `backend/.env.example`
- Add the field to `config.py` (Step 2 below will ensure it's correct either way)

**2. Get the team's Supabase connection string.**

The connection string must use the **Shared Pooler** (IPv4 compatible):
`postgresql://postgres.XXXXX:PASSWORD@aws-1-us-east-1.pooler.supabase.com:5432/postgres`

To get it: Supabase Dashboard → team project → Settings → Database → **Shared Pooler** (Session mode, port 5432). Do not use the Direct Connection — it is IPv6 only.

> **Don't have access to the Supabase dashboard?** DM Ross or Kevin in Engineering on Slack — they'll get you the connection string.

**3. Check PROJECT_CODE is unique in the team database.**

Run this Python snippet from the project root (replace the connection string with the one provided):

```python
import re, sys
from sqlalchemy import create_engine, text

conn_str = "postgresql://postgres.XXXXX:PASSWORD@aws-1-us-east-1.pooler.supabase.com:5432/postgres"
with open("backend/.env") as f:
    schema = re.search(r'PROJECT_CODE=(.+)', f.read()).group(1).strip().lower()

engine = create_engine(conn_str)
with engine.connect() as conn:
    row = conn.execute(text("SELECT 1 FROM information_schema.schemata WHERE schema_name = :s"), {"s": schema}).fetchone()
    if row:
        print(f"CONFLICT: schema '{schema}' already exists. Choose a different PROJECT_CODE.")
        sys.exit(1)
    print(f"OK: '{schema}' is available.")
```

If it exits with CONFLICT, update `PROJECT_CODE` in `backend/.env` and re-run before continuing.

---

## Step 1 — Update `backend/.env`

Add the Supabase connection string. Comment out the SQLite line. Ensure `PROJECT_CODE` is set.

```
SQLALCHEMY_DATABASE_URI=postgresql://postgres.XXXXX:PASSWORD@aws-1-us-east-1.pooler.supabase.com:5432/postgres
PROJECT_CODE=SCSDB
# SQLALCHEMY_DATABASE_URI=sqlite:///./app.db
```

Do not commit `backend/.env` — it contains the database password.

---

## Step 2 — Update source files for schema isolation

Apply the following changes to all four files. These are safe to apply whether the app was recently scaffolded or has been running for a while.

**`backend/src/config.py`** — add `PROJECT_CODE` field and `APP_SCHEMA` computed property. Keep all existing fields; only add what's missing:

```python
from pydantic import computed_field

# Inside the Settings class:
PROJECT_CODE: str = ""

@computed_field
@property
def APP_SCHEMA(self) -> str:
    return self.PROJECT_CODE.lower() if self.PROJECT_CODE else ""
```

**`backend/src/database/session.py`** — replace the engine creation with the schema-aware version. Keep the `get_db()` function if it exists:

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from src.config import settings

_is_postgres = not settings.SQLALCHEMY_DATABASE_URI.startswith("sqlite")
_kwargs: dict = {"pool_pre_ping": True}
if _is_postgres and settings.APP_SCHEMA:
    _kwargs["connect_args"] = {"options": f"-csearch_path={settings.APP_SCHEMA},public"}

engine = create_engine(settings.SQLALCHEMY_DATABASE_URI, **_kwargs)
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)
```

**`backend/src/database/base.py`** — update `Base` to use schema-aware `MetaData`. Keep all model import lines at the bottom:

```python
from sqlalchemy import MetaData
from sqlalchemy.orm import declarative_base
from src.config import settings

_schema = settings.APP_SCHEMA or None
Base = declarative_base(metadata=MetaData(schema=_schema))

# existing model imports stay here — do not remove them
```

**`backend/src/database/migrations/env.py`** — replace `run_migrations_online()` entirely. Do not touch `run_migrations_offline()` or anything outside this function:

```python
def run_migrations_online() -> None:
    from sqlalchemy import text
    from src.config import settings

    SCHEMA = settings.APP_SCHEMA

    def include_name(name, type_, parent_names):
        if type_ == "schema":
            return name == SCHEMA
        return True

    cfg = config.get_section(config.config_ini_section, {})
    cfg["sqlalchemy.url"] = settings.SQLALCHEMY_DATABASE_URI

    connectable = engine_from_config(cfg, prefix="sqlalchemy.", poolclass=pool.NullPool)

    with connectable.connect() as connection:
        if SCHEMA:
            connection.execute(text(f"CREATE SCHEMA IF NOT EXISTS {SCHEMA}"))
            connection.execute(text(f"SET search_path TO {SCHEMA}, public"))
            connection.commit()
            context.configure(
                connection=connection,
                target_metadata=target_metadata,
                version_table_schema=SCHEMA,
                include_schemas=True,
                include_name=include_name,
            )
        else:
            context.configure(
                connection=connection,
                target_metadata=target_metadata,
            )
        with context.begin_transaction():
            context.run_migrations()
```

---

## Step 3 — Apply migrations

```bash
cd backend && ../.venv/bin/python -m alembic upgrade head
```

---

## Step 4 — Migrate data (skip if no app.db)

Check whether `backend/app.db` exists. If not, skip to Step 5.

Add `migrate_to_supabase.py` to `.gitignore` first, then generate `backend/migrate_to_supabase.py`:

```python
import os
import sys
sys.path.insert(0, ".")

from dotenv import load_dotenv
load_dotenv()

from sqlalchemy import create_engine, text, MetaData

SQLITE_URL = "sqlite:///./app.db"
DB_URL = os.environ["SQLALCHEMY_DATABASE_URI"]
SCHEMA = os.environ["PROJECT_CODE"].lower()

sqlite_engine = create_engine(SQLITE_URL)
supabase_engine = create_engine(DB_URL)

meta = MetaData()
meta.reflect(bind=sqlite_engine)

with sqlite_engine.connect() as src, supabase_engine.connect() as dst:
    for table in meta.sorted_tables:
        if table.name == "alembic_version":
            continue
        rows = src.execute(text(f'SELECT * FROM "{table.name}"')).mappings().all()
        if not rows:
            print(f"Skipped {table.name} (empty)")
            continue
        cols = list(rows[0].keys())
        col_list = ", ".join(f'"{c}"' for c in cols)
        placeholders = ", ".join(f":{c}" for c in cols)
        dst.execute(
            text(f'INSERT INTO "{SCHEMA}"."{table.name}" ({col_list}) VALUES ({placeholders}) ON CONFLICT DO NOTHING'),
            [dict(r) for r in rows],
        )
        dst.commit()
        print(f"Migrated {len(rows)} rows -> {table.name}")

print("Done.")
```

Run it from `backend/`:
```bash
cd backend && ../.venv/bin/python migrate_to_supabase.py
```

Delete the script after it runs.

---

## Step 5 — Archive SQLite

Rename `backend/app.db` → `backend/app.db.bak`. Add `*.db.bak` to `.gitignore`.

---

## Step 6 — Commit

Commit these files with message `migrate: connect to team Supabase DB`:
- `backend/src/config.py`
- `backend/src/database/session.py`
- `backend/src/database/base.py`
- `backend/src/database/migrations/env.py`
- `backend/.env.example`
- `.gitignore`

Do NOT commit `backend/.env` — it contains the database password.

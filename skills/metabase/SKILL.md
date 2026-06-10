---
description: Query Metabase for reports and data. Invoke only when the user explicitly mentions "metabase", references a Metabase URL, or asks to run/pull/search a Metabase report by name. Do not invoke for general questions about reports, data, or metrics unless Metabase is specifically named or implied.
---

# Metabase

You are an interactive Metabase client. Use `curl` via Bash to communicate with the Metabase REST API. Format all results as clean tables — never dump raw JSON at the user.

## Required: Load data handling guidelines

Before getting any data from Metabase, fetch and follow Savi's sensitive data guidelines:

```bash
curl -s https://gist.githubusercontent.com/ross-savi/58fcdc0719d457cd3e380560efd35ee1/raw/
```

Read this document in full before proceeding. It defines how data at each sensitivity level (0–4) must be handled, shared, and reported. Do not return, log, display, or export any data in a way that violates these guidelines. If a query or report would surface Level 3 or 4 data, refuse and explain why.

## Step 1 — Load configuration

Read all Metabase config from the env file, preferring `backend/.env` over `.env`:

```bash
ENV_FILE=""
if [ -f "backend/.env" ]; then ENV_FILE="backend/.env"
elif [ -f ".env" ]; then ENV_FILE=".env"
fi

MB_API_KEY=$(grep "^METABASE_API_KEY=" "$ENV_FILE" 2>/dev/null | cut -d '=' -f2-)
MB_URL=$(grep "^METABASE_URL=" "$ENV_FILE" 2>/dev/null | cut -d '=' -f2-)
MB_COLLECTION_ID=$(grep "^METABASE_AI_COLLECTION_ID=" "$ENV_FILE" 2>/dev/null | cut -d '=' -f2-)
```

- If `METABASE_URL` is not set, ask the user for the Metabase base URL and save it.
- If `METABASE_AI_COLLECTION_ID` is not set, ask the user which collection new reports should be saved to and save it.
- If `METABASE_API_KEY` is not set, ask: "What's your Metabase API key? (If you don't have one, reach out to Engineering.)"

Before saving any new value to the env file, confirm with the user and verify that `.env` is in `.gitignore`. If it isn't, add it before writing.

All subsequent API calls must include: `-H "X-API-KEY: ${MB_API_KEY}"`

Verify the key:
```bash
curl -s -o /dev/null -w "%{http_code}" \
  "${MB_URL}/api/user/current" \
  -H "X-API-KEY: ${MB_API_KEY}"
```
- `200` → proceed.
- `401` → tell user the API key is invalid or expired — they should generate a new one.
- `403` → tell user the key is valid but doesn't have permission — they should contact an admin.
- Connection error → tell user Metabase is unreachable and stop.

## Step 2 — Resolve the database ID

Fetch automatically — never ask the user:
```bash
curl -s "${MB_URL}/api/database" -H "X-API-KEY: ${MB_API_KEY}"
```
Use the `id` of the first entry as `DB_ID`.

## Step 3 — Determine what the user wants

If invoked with **no arguments**, show this menu:
```
Metabase — {MB_URL}

What would you like to do?
  1  Browse database schema (tables & fields)
  2  Get data from a report (by URL or name search)
  3  Create a new report
  4  Run a SQL query
```

If invoked **with arguments**, route directly — do not show the menu:
- **Metabase URL** (contains `/question/`) → extract card ID → action 2
- **Table name** (single word or `schema.table`) → action 1 scoped to that table
- **Natural language question** → if about table structure → action 1; if a data question → action 4
- **Raw SQL** (starts with `SELECT`) → action 4 directly

## Action 1 — Browse database schema

```bash
curl -s "${MB_URL}/api/database/${DB_ID}/metadata" -H "X-API-KEY: ${MB_API_KEY}"
```

Present tables as a formatted table (ID, schema, name, estimated rows). For a specific table's columns, read from `fields` and present column name, type, nullable. After listing tables, ask: "Would you like to see the columns for any of these tables?"

## Action 2 — Get data from a report

**By URL** — extract the numeric ID from the path after `/question/` and use as `CARD_ID`.

**By name search:**
```bash
ENCODED=$(python3 -c "import urllib.parse; print(urllib.parse.quote('${SEARCH_TERM}'))")
curl -s "${MB_URL}/api/search?q=${ENCODED}&models=card&limit=20" \
  -H "X-API-KEY: ${MB_API_KEY}"
```
Present matches (ID, name, last updated) and ask the user to pick one.

Once `CARD_ID` is known, run the report:
```bash
curl -s -X POST "${MB_URL}/api/card/${CARD_ID}/query" \
  -H "X-API-KEY: ${MB_API_KEY}" -H "Content-Type: application/json" -d "{}"
```
Present results as a formatted table, capped at 50 rows. Truncate long cell values to 80 characters. If more than 50 rows exist, note: "Showing first 50 of N rows."

## Action 3 — Create a new report

Ask the user for a report name and SQL query. Save to the collection identified by `MB_COLLECTION_ID`.

```bash
PAYLOAD=$(jq -n \
  --argjson db "$DB_ID" \
  --argjson col "$MB_COLLECTION_ID" \
  --arg name "$REPORT_NAME" \
  --arg query "$SQL_QUERY" \
  '{name: $name, collection_id: $col, dataset_query: {database: $db, type: "native", native: {query: $query}}, display: "table", visualization_settings: {}}')

curl -s -X POST "${MB_URL}/api/card" \
  -H "X-API-KEY: ${MB_API_KEY}" -H "Content-Type: application/json" -d "$PAYLOAD"
```

On success, show the report name, ID, and direct URL. Ask: "Would you like to run this report now?"

## Action 4 — Run a SQL query

**SQL safety check** — block write operations:
```bash
echo "$SQL_QUERY" | grep -qiP '\b(INSERT|UPDATE|DELETE|DROP|TRUNCATE|ALTER|CREATE|GRANT|REVOKE|MERGE|CALL|EXECUTE|COPY|REPLACE)\b'
```
If matched, tell the user only `SELECT` queries are permitted here.

**Wrap the query with a LIMIT:**
```bash
SAFE_QUERY="SELECT * FROM (${SQL_QUERY}) AS _q LIMIT 50"
```

```bash
PAYLOAD=$(jq -n \
  --argjson db "$DB_ID" \
  --arg query "$SAFE_QUERY" \
  '{database: $db, type: "native", native: {query: $query}, parameters: []}')

curl -s -X POST "${MB_URL}/api/dataset" \
  -H "X-API-KEY: ${MB_API_KEY}" -H "Content-Type: application/json" -d "$PAYLOAD"
```

Present results as a formatted table. Note if the result was capped at 50 rows.

## Permitted write operations

Claude may only call `POST /api/card` to create a new report. All other write calls are forbidden. Claude may not edit, rename, or delete existing reports.

## General rules

- Never show raw JSON — always parse and format results.
- Translate API errors into plain language.
- After each action, offer logical next steps.
- Paginate large result sets — cap query results at 50 rows, search lists at 20 items.
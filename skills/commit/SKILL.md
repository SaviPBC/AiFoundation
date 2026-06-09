---
description: Run quality gates and commit changes to git. Invoke when the user says "commit", "push my changes", "ship it", or similar.
---

# Commit

## Find the project

Check that `PROJECT.md` exists in the current directory. If not, stop and ask the user to `cd` into the project root.

## Sync with remote

`git fetch origin` — pull if behind. If no remote is set, skip silently.

## Quality gate — loop until both pass, or stop after 2 fix attempts

1. Run backend tests: `cd backend && ../.venv/bin/python -m pytest src/tests/ -v`
2. Run frontend tests: `cd frontend && yarn test`
3. Review `git diff HEAD` against all of the following. Return PASS or FAIL with a bullet list of any issues found.

   **General**
   - Backend additions have unit tests
   - Frontend uses Quasar classes with minimal custom CSS
   - Code follows existing style and structure — no duplication or spaghetti code
   - No hardcoded API keys, connection strings, or secrets — must use environment variables. Exception: `SUPABASE_URL` and `SUPABASE_PUBLISHABLE_KEY` in `config.py` are intentionally baked in.

   **Security**
   - No SQL injection, XSS, or other OWASP top-10 vulnerabilities

   **Layout**
   - Any new frontend layout is justified — prefer extending existing layouts

   **Database migrations**
   - No migration whose `downgrade()` is `pass` or not implemented
   - No migration that drops or alters something in `upgrade()` without restoring it in `downgrade()`
   - No migration that drops a column or table in the same migration that creates its replacement — must be two separate migrations

   **API & data handling**
   - No external API calls without a timeout
   - No outbound requests without retry logic for transient, idempotent failures
   - No large datasets fetched in a single unbounded request — must use pagination or batching
   - No read-heavy, low-volatility, non-user-specific responses fetched repeatedly without caching
   - No sensitive data (PII, tokens, passwords) logged, stored unencrypted, or returned to the frontend unnecessarily
   - No missing error handling for failed or slow external calls
   - No user-provided credentials used to authenticate with any external service
   - No external API integrations missing a daily call limit (hard cap: 1000 calls/day per service, tracked in DB, reset midnight UTC)

4. If tests fail or review finds issues: fix them and go back to step 1.
5. After 2 failed fix attempts: stop and show the user a full report of what's failing. Do not commit.

## Commit

- Commit message format: `type(scope): description` — e.g. `feat(todos): add due date field`, `fix(api): handle empty response from payments`, `chore(deps): bump fastapi`. Keep the description under 72 characters.
- Stage tracked files: `git add -u`. Confirm before adding untracked files by listing them and asking: "These files are new and untracked — should I include them in this commit?" (`git add -u` is used here, not `git add -A`, because untracked files may be intentionally excluded.)
- Commit, then push. If no upstream: `git push -u origin <branch>`
- Never amend previous commits.

## After pushing

Review `git diff HEAD~1 HEAD` against open tickets in `PROJECT.md`. Mark any ticket whose ACs are fully satisfied as `done`. Report which tickets were closed.

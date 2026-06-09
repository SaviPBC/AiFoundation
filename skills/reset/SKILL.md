---
description: Revert all uncommitted changes back to the last git commit. Invoke when the user says "reset", "undo my changes", "go back to last commit", or similar.
---

# Reset

## Confirm first

1. Run `git status` and `git diff HEAD` to understand what will be lost.
2. Explain in plain language: which files will be reverted, whether any migrations will be rolled back and what data that means losing, that this cannot be undone.
3. Ask: "This will throw away all changes since your last commit. Any data you entered in new features will also be lost. Are you sure?"
4. Stop and wait for explicit confirmation.

## Reset

```bash
git reset --hard HEAD
git clean -fd
```

## Roll back the database

Check the migration history since the last commit:
```bash
cd backend && ../.venv/bin/python -m alembic history
```

If the migration graph since the last commit is linear, downgrade one step per migration added since the commit:
```bash
cd backend && ../.venv/bin/python -m alembic downgrade -1
```

If the migration graph is non-linear (merge commits, branching chains, or migrations that depend on each other in a non-obvious way), **stop and ask the user before downgrading** — explain what you found and what the options are.

## Wrap up

Tell the user the reset is complete. The servers keep running — they can refresh their browser. If servers stopped, give restart commands.
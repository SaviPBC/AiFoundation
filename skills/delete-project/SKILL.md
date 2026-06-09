---
description: Permanently delete a Savi project and all its files. Invoke when the user says "delete the project", "remove this project", or similar. Always confirm before proceeding.
---

# Delete Project

## Confirm first

Tell them exactly what will be deleted: running server processes, the entire project directory including the SQLite database. Warn: "This is permanent and cannot be undone. Are you sure?" Stop and wait for explicit confirmation.

## Delete

Kill servers (Mac/Linux):
```bash
lsof -ti:8000 | xargs kill -9 2>/dev/null || true
lsof -ti:9000 | xargs kill -9 2>/dev/null || true
```

Delete the project folder (from one level up):
```bash
rm -rf <project-directory>
```

Tell the user the project is removed and remind them to manually delete any remote repo if one exists.
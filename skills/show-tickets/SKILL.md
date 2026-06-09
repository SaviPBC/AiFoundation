---
description: Display the current ticket list from PROJECT.md. Invoke when the user says "show tickets", "what are we working on", "ticket status", or similar.
---

# Show Tickets

Check that `PROJECT.md` exists in the current directory. If not, stop and ask the user to `cd` into the project root.

Read `PROJECT.md` and display tickets grouped by status:

**In Progress → Backlog (by priority) → Done**

For each ticket: number, title, assignee, blockers. Done tickets can be collapsed to a count unless the user asks for details.
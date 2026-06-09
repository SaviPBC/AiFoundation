---
description: Act as product manager to plan features, update the roadmap, or add/change tickets. Invoke when the user says "plan features", "add a ticket", "what's the plan", "update the roadmap", or "product manager".
---

# Manage Tickets

Act as the product manager. Your source of truth is `PROJECT.md`. If it doesn't exist, create it.

**If no specific input is given:**
- Read `PROJECT.md`
- Summarize: current roadmap phase, tickets done/in-progress/backlog, next highest-priority ticket
- Ask what they'd like to work on or change

**Updating the project:**
- Translate vague or non-technical input into clear, actionable tickets
- If a request is large or spans multiple concerns, split it into focused tickets (see **Breaking down large requests** below)
- Make reasonable software decisions without asking — surface assumptions at the end
- Ask one clarifying question if scope is genuinely unclear, then proceed
- Write the updated `PROJECT.md` to disk after every change — don't ask permission
- Present new/changed tickets clearly: number, title, status, blockers
- Ask: "Does this look right? Want to adjust anything before we start building?"

## PROJECT.md format

Maintain this exact structure:

```markdown
# Project Vision

One short paragraph describing the goal and purpose of the app.

---

# Features

- **Feature Name** — brief description of what it does for the user

---

# Pages

- **Page Name** (`/route`) — what the user does here

---

# Roadmap

- **Phase 1: Name** — what gets built in this phase
- **Phase 2: Name** — ...

---

# Version

Current semver version. Bump minor (0.x.0) when closing a roadmap phase.

---

# Config

- `PROJECT_CODE`: `VALUE` — short all-caps identifier (5–7 chars, 10 max) unique to this app, used as its Postgres schema name when graduated to a team database

---

# Database Glossary

- **table_name** — what it stores, key fields, notable relationships

---

# Tickets

Tickets ordered by priority (highest first).

---

### PE-001 · Ticket Title

**Status:** `backlog` | `in-progress` | `done`
**Assigned to:** Claude | User
**Blocked by:** PE-XXX  *(omit if not blocked)*

**Description:**
What needs to be built and why.

**Acceptance Criteria:**
- [ ] AC 1
- [ ] AC 2
```

## Ticket rules

- **Split on real boundaries, not file boundaries.** A ticket is a meaningful deliverable unit — not a single file or step. Split when pieces require different timing or when one genuinely cannot start until another is deployed.
  - ✗ "Build page.vue" + "Register the route" → one ticket: "Add /foo page"
  - ✓ "Add data model + migration" + "Build API endpoint" + "Build UI" → split because each layer depends on the previous
- Each AC is testable and concrete — not a task description.
- Assign to Claude for engineering work, to User for decisions requiring real-world info or credentials.
- A ticket is blocked only when it literally cannot start without the blocker being done.
- Never reuse ticket numbers. Read the existing highest number before creating new ones.
- Bug fixes and blockers jump to the top of the backlog.

## Breaking down large requests

When a user request is large enough that a single ticket would be unwieldy (more than ~5 ACs, multiple layers, or touching more than ~10 files), propose a **phased sequence** of tickets instead.

**Phase order:**
1. **Data layer** — models, migrations, seed data
2. **Service/API layer** — endpoints, business logic, CRUD
3. **UI layer** — pages, components, state
4. **Polish** — validation, error states, empty states, loading indicators

Each phase is a ticket; later phases depend on earlier ones via `Blocked by:`.

**Always propose the breakdown to the user before creating the tickets.** Only create them after the user confirms. Do not silently split or silently merge.

## Prioritization rules

- Infrastructure and data model tickets come before features that depend on them.
- Core user-facing flows come before nice-to-haves.
- Bug fixes and blockers jump to the top.
- When in doubt, ask the user which outcome matters most right now.
- Bump minor version in `VERSION` and `frontend/package.json` when closing a roadmap phase.
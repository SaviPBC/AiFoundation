---
description: Implement a ticket from PROJECT.md. Invoke when the user says "work on the next ticket", "build this", "code", "implement", or references a specific ticket number.
---

# Build a Ticket

Implement exactly what the ticket asks — no more, no less.

## Find the project

Check that `PROJECT.md` exists in the current directory. If it doesn't, stop and ask the user to `cd` into the project root.

## Reconcile PROJECT.md

Before starting new work, quickly check whether `PROJECT.md` reflects reality:
- If tickets are marked `backlog` but the feature is clearly already built, update them to `done`.
- If tickets are marked `done` but the code was reverted, update them back to `backlog`.
- Do this silently — only mention it if you find and fix a discrepancy.

## Sync with remote

`git fetch origin` — if behind, `git pull` automatically. If uncommitted changes would conflict, tell the user to commit first and stop.

## Pick the ticket

- If the user named a ticket number, use that ticket.
- If the user provided feedback alongside a ticket (e.g. "the button doesn't show on mobile"), treat that feedback as the specific problem to fix — do not re-implement the whole ticket, focus only on what the feedback describes.
- If the user reported a bug or error with no ticket reference, create a ticket for it in `PROJECT.md` first, then work on that ticket.
- Otherwise, pick the highest-priority unblocked ticket from `PROJECT.md`.
- If the top ticket is blocked, explain why in plain language and tell the user what to do next — either work the blocking ticket first or adjust the plan via the manage-tickets skill.

## Before writing any code

1. Read `PROJECT.md` and find the ticket. Understand the project vision and how this ticket fits.
2. Read all existing code you plan to touch — never modify code you haven't read.
3. If requirements are genuinely ambiguous, ask one focused question and wait for the answer.
4. If completing the ticket correctly requires additional ACs that are missing, propose them and update the ticket in `PROJECT.md` before proceeding.

## Writing code

- Do not ask permission — just make the changes.
- Follow existing patterns — match naming, structure, and style.
- Do not refactor surrounding code or add features beyond the ticket's ACs.
- Do not add comments unless the logic is genuinely non-obvious.
- Apply all rules from the `savi-ai-foundation:guardrails` skill: tests, security, environment variables, migrations, API rules, analytics.

## After writing code

- Update the ticket status to `done` in `PROJECT.md`. Do not ask permission.
- Bump **patch** (0.0.x) in `VERSION` and `frontend/package.json`.
- If you cannot complete the ticket, follow the **When Things Break** pattern from the guardrails skill.
- End with a plain-language summary of what changed, which files were touched, and how to verify it manually.
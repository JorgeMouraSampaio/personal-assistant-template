---
name: obsidian-create-task
description: >-
  Create tasks in this Obsidian vault's Tasks base from a natural-language
  description of what the user needs to do. Use whenever the user describes
  something they need to get done — "I need to…", "remind me to…", "add a task
  to…", "I have to send X to Y by Friday" — and expects it captured as a task.
  Parses the due date, stakeholders (people), company, and project out of the
  sentence, auto-assigns a task number, creates the task note, and
  crosslinks it. Tasks appear automatically on the kanban boards (TaskNotes
  views) and in their project's dashboard. This is task-entry METHOD 1 (Claude
  creates the task from your spoken/typed request); other entry methods may be
  added later.
---

# Create Task (Method 1 — from natural language)

The user describes work in plain language; you turn it into properly-structured
task notes in the Tasks base. Read the `obsidian-base-cleanup` skill for the
wider vault model (tags, wikilink quoting, `People/<Company>/` folders) — this
skill assumes it.

**No validation gate here.** Asking for a task IS the validation, so METHOD 1
creates the note straight away. Tasks mined from a meeting (METHOD 2,
`obsidian-process-meeting`) are different: they are proposed to the user and only
written once he signs off. That skill owns the gate; this one is the schema and
numbering it borrows.

## Task note schema

Tasks are notes tagged `Task`, all filed **flat in `Tasks/Task Notes/`** — there
are no per-project subfolders. The `Project` frontmatter (a wikilink, or plain
`General task`) is how the Tasks base organizes tasks; the folder plays no
organizing role. `General task` is a plain-text placeholder for a task with no
project — it is **never a real project note and must never be wikilinked**
(`Project: General task` stays plain text). Frontmatter:

```yaml
---
tags:
  - Task
Task Number: "1"             # ALWAYS quoted — keeps the property a string, not a number
Status: Not Started          # Not Started | In Progress | Completed | Discarded
Project: General task        # plain text "General task", or "[[Project Name]]"
Stakeholders:                # list of "[[Person]]" links (people mentioned)
  - "[[Jane Doe]]"
Company: "[[Acme]]"          # "[[Company]]" wikilink, or blank
Created Date: 2026-01-01     # today, unquoted ISO date
Due Date:                    # ISO date if one is implied, else blank
Priority:                    # High | Medium | Low — optional, blank unless the user signals importance
Effort:                      # estimated minutes, multiples of 15 (15,30,45,…); blank if unknown
Booked ?: N                  # Y once a calendar slot is booked for it, else N (default N)
Notes: free text (may contain [[links]])
Links:                       # URL(s) mentioned for the task (doc, ticket, etc.), else blank
---
```

**Filename** = the short title only, e.g. `Send invoice.md` — do NOT prefix the
number (the number is its own `Task Number` property/column). Wikilinks therefore
target the plain title: `[[Send invoice]]`.

## Numbering

- Task numbers are plain whole numbers: `1`, `2`, `3`. Assign the next free
  number = (highest existing integer across `Tasks/`) + 1.
- Before creating, scan existing `Tasks/**/*.md` `Task Number` values to find the
  next number — don't assume the vault is empty.
- There is no sub-task hierarchy: every task is a flat, standalone note. If a
  request describes prerequisite steps, either fold them into `Notes` or create
  separate sibling tasks and cross-reference them in `Notes`
  ("blocked by [[other task]]").

## How to parse a request

Pull these out of the sentence; leave blank what isn't stated (don't invent):

- **Title** — the core action, imperative and short ("Send invoice").
- **Due Date** — resolve relative dates against today's date from context
  ("tomorrow", "by Friday", "next week") into an absolute `YYYY-MM-DD`.
- **Stakeholders** — every person named. Each becomes a `[[Person]]` link. If a
  person has no note yet, **create it** under `People/<Company>/` (follow
  `obsidian-base-cleanup`); infer their company from context (internal colleagues
  named with roles like CFO/HR → your own org).
- **Company** — the client/company the work concerns. If the stakeholders are
  internal staff and no client is named, use your own org; if a client is named,
  link it; if genuinely none, leave blank.
- **Project** — link `[[Project]]` if the work clearly belongs to a known project;
  otherwise `Project: General task`. Tasks need not have a project.
- **Links** — any URL(s) in the request (doc, ticket, etc.) go in `Links`, verbatim.

## Status & kanban boards

**The note's `Status` frontmatter is the single source of truth.** The kanban
boards — `Tasks/Tasks Kanban.base` (global) and the Board embedded in each
project dashboard — are **TaskNotes views** over that property: a new
`Not Started` task appears in the right column by itself, and dragging a card
writes the new `Status` back into the note. There is no board file to edit and
no card to move — to change a task's status, change its `Status` property (or
let the user drag the card), nothing else.

## Verify
After creating, confirm no broken wikilinks (see the check in
`obsidian-base-cleanup`) — task↔task, task→person, and task→company links must all
resolve.

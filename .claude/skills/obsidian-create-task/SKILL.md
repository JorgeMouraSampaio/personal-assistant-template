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
  views) and on their project's or initiative's dashboard. This is task-entry METHOD 1 (Claude
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
are no per-project subfolders. The `Project` and `Initiative` frontmatter is how
the Tasks base organizes tasks; the folder plays no organizing role. Frontmatter:

```yaml
---
tags:
  - Task
Task Number: "1"             # ALWAYS quoted — keeps the property a string, not a number
Status: Not Started          # Not Started | In Progress | Blocked | Completed | Discarded
Project: General task        # "[[Project Name]]", or plain "General task" / "Personal Task", or blank
Initiative:                  # "[[Initiative Name]]" when the task's container is an initiative, else blank
Stakeholders:                # list of "[[Person]]" links (people mentioned)
  - "[[Jane Doe]]"
Company: "[[Acme]]"          # "[[Company]]" wikilink, or blank
Created Date: 2026-01-01     # today, unquoted ISO date
Due Date:                    # ISO date if one is implied, else blank
Priority:                    # High | Medium | Low — optional, blank unless the user signals importance
Effort:                      # estimated minutes, multiples of 15 (15,30,45,…); blank if unknown
Booked ?: N                  # Y once a calendar slot is booked for it, else N (default N)
Notes: "free text (may contain [[links]])"   # always double-quoted, see CLAUDE.md rule 1
Links:                       # URL(s) mentioned for the task (doc, ticket, etc.), else blank
---
```

`Initiative` sits immediately after `Project`. Keep that order.

A Blocked task also carries `Blocker: "<one sentence>"`, quoted, placed right
after `Status`: what the task waits on and who owns it. Add it only when the user
says the task is blocked, in his words. Clear it to blank when the task leaves
`Blocked`. A new task starts `Blocked` only when the user says so at creation.

**Filename** = the short title only, e.g. `Send invoice.md` — do NOT prefix the
number (the number is its own `Task Number` property/column). Wikilinks therefore
target the plain title: `[[Send invoice]]`.

## The container: project, initiative, or neither

Every task has exactly one container, and `Project`/`Initiative` are mutually
exclusive. Pick one of four shapes:

| The work belongs to | `Project` | `Initiative` |
|---|---|---|
| a project | `"[[Widget]]"` | blank |
| an initiative (see `obsidian-base-cleanup`) | **blank** | `"[[Widget Launch]]"` |
| no container, work task | `General task` (plain text) | blank |
| no container, the user's own to-do | `Personal Task` (plain text) | blank |

`General task` and `Personal Task` are plain-text placeholders, **never real
notes and never wikilinked**. `Personal Task` always pairs with
`Company: N/A` (plain text). Never fill both `Project` and `Initiative`: a task
under a project is already under that project's initiative, through the project.

**The TaskNotes caveat.** TaskNotes maps exactly one property as its project
field (`Project`), so an initiative-only task shows a **blank project** in
TaskNotes' own UI and in the kanban boards' project field. That is cosmetic and
expected: don't "fix" it by duplicating the initiative into `Project`. Every
Bases view still picks the task up, because those views filter on
`file.hasLink(this.file)`, which matches a link in any frontmatter property.

## Register the task on its container

A task↔container relationship is written on **both** sides. After creating the
task note, append it to the container note's `Related Tasks` list, in the aliased
form that displays the number:

```yaml
Related Tasks:
  - "[[Send invoice|#195]]"
```

Append to the end of the existing list (the list reads in task-number order), and
create the key's list if the container had none. Both project and initiative
notes carry `Related Tasks`. A task whose `Project` is `General task` or
`Personal Task` has no container, so nothing is appended.

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
- **Company** — the client/company the work concerns. A task under a project or
  an initiative inherits that container's `Company`. Otherwise: if the
  stakeholders are internal staff and no client is named, use your own org; if a
  client is named, link it; if genuinely none, leave blank.
- **Container** — set `Project: "[[Project]]"` when the work clearly belongs to a
  known project. When it belongs to a standing track rather than a bounded
  project, leave `Project` blank and set `Initiative: "[[Initiative]]"` instead.
  Failing both, `Project: General task`, or `Personal Task` for the user's own
  errands and appointments (with `Company: N/A`). **Never create an initiative or
  a project just to hold a task**: link one that exists, or fall back to
  `General task`.
- **Links** — any URL(s) in the request (doc, ticket, etc.) go in `Links`, verbatim.

## Status & kanban boards

**The note's `Status` frontmatter is the single source of truth.** The kanban
boards — `Tasks/Tasks Kanban.base` (global) and `All Tasks.base#Project Board` —
are **TaskNotes views** over that property: a new `Not Started` task appears in
the right column by itself, and dragging a card writes the new `Status` back into
the note. There is no board file to edit and no card to move — to change a task's
status, change its `Status` property (or let the user drag the card), nothing
else. Project and initiative dashboards embed the **table**
(`![[All Tasks.base#Project Table]]`), not the board, so a new task lands there
by itself too.

## Verify
After creating, confirm no broken wikilinks (see the check in
`obsidian-base-cleanup`) — task↔task, task→person, task→company, and
task↔container links must all resolve. Check the aliased `Related Tasks` entry
too: the part **before** the `|` is the link target, so a typo there breaks the
link while still displaying a tidy `#195`.

---
name: obsidian-log-session
description: >-
  Close out a work session on a project or initiative in this Obsidian vault by
  writing a crosslinked session note into Sessions/<Project or Initiative>/. Use
  whenever the user says
  "end/finish/close/wrap up the session", "log this session", "I'm done for
  today/now", "session summary", or otherwise signals they are stopping work on
  a project. The session note documents what was done, decisions made, which
  tasks were completed and which are still open; it updates task Statuses first,
  appends decisions to the container dashboard's Key Decisions, and
  shows up automatically in the Sessions base and that dashboard's
  Sessions view.
---

# Log Session (close out a work session → session note)

When the user finishes working on a project or an initiative, this skill captures
the session as a permanent, crosslinked record. Session notes are what make a
dashboard's **Sessions** view and the vault-wide **Sessions base** work — each
note is one row.

Read `obsidian-base-cleanup` (vault model, wikilink quoting) first; compose
`obsidian-create-task` if new follow-up tasks surface while closing out.

## Session note schema

Sessions are notes tagged `Session`, filed into
**`Sessions/<Project or Initiative>/`** — the folder is named after whichever
container the session's work sat under (use `Sessions/General/` when it had
neither). Filename = `YYYY-MM-DD <Project or Initiative> Session` (append ` 2`,
` 3`… for multiple same-day sessions) — date-prefixed so basenames stay unique
vault-wide.

```yaml
---
tags:
  - Session
Date: 2026-07-04            # the day the session happened
Project: "[[Widget]]"        # or plain blank; never wikilink "General task"
Initiative:                  # "[[Widget Launch]]" when the container is an initiative, else blank
Company: "[[Acme]]"          # inherited from the project or initiative, or blank
Tasks Completed:             # tasks whose Status became Completed this session
  - "[[Send invoice]]"
Tasks In Progress:           # tasks touched but still open
  - "[[Draft proposal]]"
---
```

`Initiative` sits immediately after `Project`, and the two are mutually
exclusive: a session on a project leaves `Initiative` blank, a session on an
initiative leaves `Project` blank. The Sessions base and the dashboards' Sessions
view both filter on `file.hasLink(this.file)`, so either key puts the session on
the right dashboard.

Body sections (all four, even if one is just "None."):

```markdown
## What was done

## Decisions

## Tasks completed

## Tasks still open
```

`Sessions/` is deliberately **excluded from Templater folder-templates** (same
reason as `Meetings/`): session notes are created fully-formed by this skill, and
a trigger-on-create template would clobber them.

## Workflow

### 1. Identify the container and what happened
The container is whatever the session's work was about, a project or an
initiative — from the conversation, or ask if genuinely ambiguous. Gather: work
performed, decisions made, tasks whose status changed, new follow-ups that came
up.

### 2. Update task Statuses FIRST
Before writing the note, bring every touched task's `Status` frontmatter up to
date (Completed / In Progress / …). **Status frontmatter is the single source of
truth** — the kanban boards are TaskNotes views over it and update themselves.
Create any new follow-up tasks per `obsidian-create-task`.

### 3. Write the session note
Create `Sessions/<Container>/YYYY-MM-DD <Container> Session.md` with the schema
above, where `<Container>` is the project or initiative basename.
- `## What was done` — short prose bullets of the session's actual work.
- `## Decisions` — every decision made, one bullet each.
- `## Tasks completed` / `## Tasks still open` — `[[task]]` links matching the
  frontmatter lists (with a word on remaining work for open ones).
- Record only what actually happened; prefer a blank over an invented value.

### 4. Append container-level decisions to the dashboard
For each decision that shapes the work going forward (not session trivia),
append to the container note's `## Key Decisions` section:
`- YYYY-MM-DD — <decision> (from [[<session basename>]])`. Initiative notes carry
the same `## Key Decisions` section as project notes.

### 5. Verify
Broken-wikilink check (`obsidian-base-cleanup`):
session→project/initiative/company, session→tasks, and the Key Decisions backlink
must all resolve. Confirm the session appears in the container dashboard's
Sessions view (it will, if `Project` or `Initiative` is a proper quoted
wikilink). Then summarize the session note back to the user in two or three
lines.

## Detailed reports / findings

If closing out produces a detailed deliverable (a review list, an audit table,
research findings…), it is **part of the session closure — there is no separate
`Reports/` folder**. Small: a section in the session note body. Large enough to
drown the note: a companion note in the same `Sessions/<Container>/` folder
(date-prefixed filename, frontmatter `Date`/`Project`/`Initiative`/`Company`
wikilinks, **no `Session` tag** — it's not a session row), linked from the
session note and from any related task's `Links`.

## Judgment & safety
- Don't log a session nobody asked to close: trigger on clear "I'm done /
  wrap up" signals, not mid-work lulls.
- One session note per sitting, not per task.
- The note is a record, not a plan — future work goes into Tasks, with at most a
  one-line pointer under `## Tasks still open`.

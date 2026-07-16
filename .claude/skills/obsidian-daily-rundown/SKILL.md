---
name: obsidian-daily-rundown
description: >-
  Generate the task rundown note in this Obsidian vault — a digest of open tasks
  bucketed by due date, written into Tasks/Rundowns/. Two layouts: a morning
  rundown (overdue / due today / this week / next week / no due date) and an
  end-of-day rundown (due tomorrow / this week / next week / no due date / all
  ongoing tasks). Use when the user asks "what's on my plate", "daily rundown",
  "on-demand rundown", "end-of-day rundown", "what's due", or when a scheduled
  run fires. Reads every task note's Status and Due Date; creates no tasks and
  changes no Statuses.
---

# Task Rundown (open tasks by due date → digest note)

A read-only digest: scan the Tasks base, bucket open tasks by `Due Date`, and
write one rundown note. Read `obsidian-base-cleanup` for the vault model
(wikilink quoting, folder layout) — this skill assumes it.

## Triggers and modes

Two scheduled jobs, plus on-demand:

- **Morning** — the `daily-task-rundown` job fires at 9am (what's due now).
- **End-of-day** — the `eod-task-rundown` job fires at 6pm (looks ahead to
  tomorrow, plus the full open list).
- **On-demand** — the user asks any time. Default to the **morning** layout,
  unless they ask for an "end-of-day rundown" (then use that layout). On-demand
  additionally presents the rundown in the chat reply (the user is waiting on it).

There is only ever **one** rundown note (see Retention); every run overwrites it
with a fresh snapshot. The `Generated` frontmatter stamps the datetime, mode, and
trigger.

## What counts (both layouts)

- Scan every `Tasks/**/*.md` tagged `Task` (skip `Tasks/Rundowns/` and the
  `.base` files).
- **Open** = `Status` is `Not Started` or `In Progress`. Completed and Discarded
  never appear.
- Weeks run **Monday–Sunday**, computed from today.
- A dated task lands in exactly **one** dated bucket — its soonest matching one
  (so a task due tomorrow is never repeated under "this week").
- Sort dated buckets by due date then Task Number; sort undated by Task Number.
- Render each task as `- [[Title]] — due YYYY-MM-DD (Ddd) · [[Project]] · Status`.
  Omit the "due …" part where the bucket already implies the date (Due today, Due
  tomorrow, No due date, and the undated rows of All Ongoing Tasks). `Project` is
  the task's value (`[[Project]]` or plain `General task`).

## Morning layout (9am job + default on-demand)

Five sections, always present (empty bucket → `- None.`):

1. **Overdue** — `Due Date` before today
2. **Due today** — `Due Date` = today
3. **Due this week** — after today through this Sunday
4. **Due next week** — next Monday through next Sunday
5. **No due date** — blank `Due Date`

Footer: `N tasks due later than next week (not listed).`

## End-of-day layout (6pm job + on-demand "end-of-day rundown")

Five sections, always present (empty bucket → `- None.`):

1. **Due tomorrow** — `Due Date` = tomorrow
2. **Due this week** — after tomorrow through this Sunday
3. **Due next week** — next Monday through next Sunday
4. **No due date** — blank `Due Date`
5. **All Ongoing Tasks** — every open task, the complete list (a superset of the
   buckets above), sorted by due date then Task Number, undated last

No footer — All Ongoing Tasks already lists everything.

## The rundown note

Path: `Tasks/Rundowns/YYYY-MM-DD Rundown.md` (today's date). If it already exists
(a scheduled run already wrote it, or a re-run), overwrite it. Not tagged `Task`
(it must never appear in the Tasks base or on kanban boards). `Generated` records
the datetime, mode, and trigger so it's clear which layout it is and how fresh.

```markdown
---
tags:
  - Rundown
Date: 2026-07-08
Generated: 2026-07-08 18:00 (end-of-day, scheduled)
---
# Task Rundown (end of day) — Wednesday, 2026-07-08

## Due tomorrow
- [[Task title]] — [[Project]] · Not Started

## Due this week
- [[Task title]] — due 2026-07-11 (Sat) · [[Project]] · Not Started

## Due next week
- [[Task title]] — due 2026-07-15 (Wed) · [[Project]] · Not Started

## No due date
- [[Task title]] — [[Project]] · Not Started

## All Ongoing Tasks
- [[Task title]] — due 2026-07-11 (Sat) · [[Project]] · Not Started
- [[Task title]] — [[Project]] · Not Started
```

(The morning layout uses the same frontmatter, heading `# Task Rundown — <Weekday>,
<date>`, its five sections, and the "due later" footer. `Generated`'s label is the
mode + trigger, e.g. `(morning, scheduled)` or `(morning, on-demand)`.)

- Every section always present; an empty bucket gets `- None.`
- **Retention:** keep only the current rundown. After writing today's, delete
  every other note in `Tasks/Rundowns/` — past rundowns have no value (they're
  disposable digests, not records), so only one ever exists at a time. (The
  morning and end-of-day runs share the same daily filename, so the later run
  simply refreshes the single note.)

## Guardrails

- Read-only with respect to tasks: never create tasks, never touch `Status` or
  any other task frontmatter from this skill.
- `Tasks/Rundowns/` is shielded from the Task folder template (a `Noop.md`
  Templater mapping plus a guard inside `Templates/Task.md`) — keep it that
  way; rundown notes are created fully-formed.
- Idempotent: running it again just rewrites today's note.

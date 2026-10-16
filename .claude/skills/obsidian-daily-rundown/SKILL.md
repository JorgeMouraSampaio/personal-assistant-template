---
name: obsidian-daily-rundown
description: >-
  Generate the task rundown note in this Obsidian vault — a digest of open tasks
  bucketed by due date into one bucket per business day for the next five business
  days (each labeled by its weekday name), plus a cross-cutting "Ongoing Tasks"
  section for everything In Progress regardless of due date, and a "Blocked"
  section that gives each Blocked task's blocker in one sentence, written into
  Tasks/Rundowns/. Two layouts: a morning rundown (ongoing tasks / blocked / overdue / due
  today / due tomorrow / then each of the next four business days by weekday / due
  later / no due date) and an end-of-day rundown (ongoing tasks and blocked first, then the same
  forward business-day buckets without "due today", plus all ongoing tasks). Use when
  the user asks "what's on my plate",
  "backlog",
  "daily rundown", "on-demand rundown", "end-of-day rundown", "what's due", or when
  a scheduled run fires. Reads every task note's Status and Due Date; creates no
  tasks and changes no Statuses.
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
  In the chat reply, render **one bullet per task** (each on its own line, led by
  its `#<Task Number>`) — never group, compress, or collapse multiple tasks onto a
  single line or into category summaries, no matter how long the list.

There is only ever **one** rundown note (see Retention); every run overwrites it
with a fresh snapshot. The `Generated` frontmatter stamps the datetime, mode, and
trigger.

## What counts (both layouts)

- Scan every `Tasks/**/*.md` tagged `Task` (skip `Tasks/Rundowns/` and the
  `.base` files).
- **Open** = `Status` is `Not Started`, `In Progress`, or `Blocked`. Completed and
  Discarded never appear.
- **Blocked** (heading `## Blocked ⛔`): every task with `Status: Blocked`, and
  **only** here. A Blocked task stays out of Ongoing Tasks, every due-date bucket,
  and No due date, since the user cannot act on it until the blocker clears. (End of
  day: it still appears in All Ongoing Tasks, which lists every open task.) Render
  each as `- #<Task Number> [[Title]] — due YYYY-MM-DD (Ddd) · [[Project]] · Blocked: <Blocker>`,
  where `<Blocker>` is the task's `Blocker` sentence verbatim. Drop the "due …" part
  when the task is undated. If `Blocker` is blank, print
  `Blocked: ⚠️ no blocker recorded` so the gap gets fixed. Sort by due date then
  Task Number, undated last. In the chat reply, keep the blocker sentence on the
  task's own bullet.
- Buckets are **rolling per-business-day windows relative to today** (`T`), one
  bucket per business day for the **next five business days**. Compute `BD1`…`BD5`
  by stepping forward from `T`, skipping Sat/Sun. From a **Wednesday**: BD1=Thu,
  BD2=Fri, BD3=Mon, BD4=Tue, BD5=Wed(next). (Five business days always spans exactly
  one week, so BD5 lands on the **same weekday as `T`**, one week out.)
- The dated buckets are contiguous, half-open, and **mutually exclusive** — a task
  lands in its soonest match. **Weekend due dates fold into the next business day's
  bucket** (a Sat/Sun task lands in that Monday bucket):
  - **Overdue** — `due < T`
  - **Due today** — `due = T`
  - **BD1** — `T < due ≤ BD1`
  - **BD2** — `BD1 < due ≤ BD2`
  - **BD3** — `BD2 < due ≤ BD3`
  - **BD4** — `BD3 < due ≤ BD4`
  - **BD5** — `BD4 < due ≤ BD5`
  - **Due later** — `due > BD5`
- **Heading labels use the weekday name of each bucket's business day**, auto-filled:
  `Due today` for `T`; for BD1 use `Due tomorrow` when BD1 is the calendar day after
  `T` (today is Mon–Thu), otherwise `Due <Weekday>` (on a Friday, BD1 is Monday →
  `Due Monday`); `Due <Weekday>` for BD2–BD5 (e.g. `Due Friday`, `Due Monday`,
  `Due Tuesday`, `Due Wednesday`). Then `Due later` (beyond BD5) and `No due date`.
- **Ongoing Tasks** — a cross-cutting section, independent of the due-date buckets
  above: every task with `Status: In Progress`, regardless of its due date (or lack
  of one). This surfaces long-running active work that isn't necessarily near due,
  so it doesn't get lost. It's additive, not exclusive — an In Progress task with a
  near due date still appears in **both** its due-date bucket above **and** here.
  Sort by due date then Task Number, undated last.
- **No due date** (heading `## No due date ⚠️`) — every open task with a blank
  `Due Date`, whether `Not Started` or `In Progress` (Blocked tasks stay in Blocked). The
  ⚠️ flags these as needing a due date. Undated **In Progress** tasks are **not**
  excluded here (unlike the older backlog rule): they'd otherwise be easy to lose
  track of, so they appear both here **and** under Ongoing Tasks. Sort by Task Number.
- Sort dated buckets by due date then Task Number; sort undated by Task Number.
- Render each task as `- #<Task Number> [[Title]] — due YYYY-MM-DD (Ddd) · [[Project]] · Status`.
  Always lead with the `#<Task Number>` (both in the note and in the on-demand chat
  reply). Omit the "due …" part only for **Due today** and **No due date**; **keep**
  it in every forward bucket (task lines carry the exact date, which also disambiguates
  BD5 from today since they share a weekday name, and the weekend-absorbing Monday
  bucket).
- The container slot is the task's `Project` (`[[Project]]`, or plain
  `General task` / `Personal Task`). **When `Project` is blank, print the
  `Initiative` instead** — a task can belong to an initiative rather than a
  project, and every bullet must show a container. Render it as
  `[[Initiative]]`, the same way. If both are blank, the task is malformed:
  print `⚠️ no container` rather than dropping the field.

## Morning layout (9am job + default on-demand)

Eleven sections, always present (empty bucket → `- None.`), in order. **Ongoing Tasks
leads** (active work first), then Blocked, then the due-date buckets:

1. **Ongoing Tasks** — every `Status: In Progress` task, cross-cutting (see above)
2. **Blocked** — heading `## Blocked ⛔`; every `Status: Blocked` task with its blocker (see above)
3. **Overdue** — `due < T`
4. **Due today** — `due = T`
5. **BD1** — heading `## Due tomorrow` (or `## Due <Weekday>` if BD1 isn't calendar-tomorrow)
6. **BD2** — `## Due <Weekday>`
7. **BD3** — `## Due <Weekday>`
8. **BD4** — `## Due <Weekday>`
9. **BD5** — `## Due <Weekday>`
10. **Due later** — `due > BD5` (heading `## Due later (> 5 business days)`)
11. **No due date** — heading `## No due date ⚠️`; every open task with a blank
    `Due Date`, regardless of Status (undated In Progress tasks appear here too, and
    also under Ongoing Tasks)

No footer (Due later is a listed section).

## End-of-day layout (6pm job + on-demand "end-of-day rundown")

Same forward per-business-day buckets, but **no "Due today"** (the day is over; any
still-open due-today task shows in All Ongoing Tasks), plus a full list at the end.
Sections, always present (empty bucket → `- None.`). **Ongoing Tasks leads**, then
the due-date buckets:

1. **Ongoing Tasks** — every `Status: In Progress` task, cross-cutting (see above)
2. **Blocked** — heading `## Blocked ⛔`; every `Status: Blocked` task with its blocker (see above)
3. **Overdue** — `due < T`
4. **BD1** — `## Due tomorrow` (or `## Due <Weekday>`)
5. **BD2** — `## Due <Weekday>`
6. **BD3** — `## Due <Weekday>`
7. **BD4** — `## Due <Weekday>`
8. **BD5** — `## Due <Weekday>`
9. **Due later** — `due > BD5`
10. **No due date** — heading `## No due date ⚠️`; every open task with a blank
   `Due Date`, regardless of Status (undated In Progress tasks appear here too, and
   also under Ongoing Tasks)
11. **All Ongoing Tasks** — every open task, the complete list (a superset of
    everything above, including due-today and due-later tasks, regardless of
    Status), sorted by due date then Task Number, undated last. (Distinct from
    "Ongoing Tasks" above, which is filtered to `In Progress` only.)

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

## Overdue
- [[Task title]] — due 2026-07-06 (Mon) · [[Project]] · Not Started
- [[Task title]] — due 2026-07-07 (Tue) · [[Initiative]] · In Progress

## Due next business day
- [[Task title]] — due 2026-07-09 (Thu) · [[Project]] · Not Started

## Due within a week
- [[Task title]] — due 2026-07-13 (Mon) · [[Project]] · Not Started

## Due within two weeks
- [[Task title]] — due 2026-07-20 (Mon) · [[Project]] · Not Started

## No due date ⚠️
- [[Task title]] — [[Project]] · Not Started

## All Ongoing Tasks
- [[Task title]] — due 2026-07-09 (Thu) · [[Project]] · Not Started
- [[Task title]] — [[Project]] · Not Started
```

(The morning layout uses the same frontmatter, heading `# Task Rundown — <Weekday>,
<date>`, and its eleven sections in the order listed above — Ongoing Tasks, Blocked ⛔, Overdue,
Due today, the five forward business-day buckets, Due later, and No due date ⚠️ —
with no footer. `Generated`'s label is the mode + trigger, e.g. `(morning, scheduled)`
or `(morning, on-demand)`.)

- Every section always present; an empty bucket gets `- None.`
- **Pending task proposals.** If `Tasks/Pending/Proposed Tasks.md` holds any proposal
  blocks, append a final `## Pending task proposals ⚠️` section (both layouts) with one
  bullet per staged proposal: `- <Task title> (from [[<meeting>]])` plus due date if
  set. This is the **only** conditional section: when the queue is empty, omit it
  entirely. Flag it in the chat rundown too, with the count and a nudge to validate
  ("3 task proposals from 2 meetings waiting on your OK"). Proposals are not tasks:
  never number them, never count them in the due-date buckets or in All Ongoing Tasks.
- **Retention:** keep only the current rundown. After writing today's, delete
  every other note in `Tasks/Rundowns/` — past rundowns have no value (they're
  disposable digests, not records), so only one ever exists at a time. (The
  morning and end-of-day runs share the same daily filename, so the later run
  simply refreshes the single note.)

## Guardrails

- Read-only with respect to tasks: never create tasks, never touch `Status` or
  any other task frontmatter from this skill. That includes the pending-proposal
  queue: this skill only reports it, never creates from it or edits it.
- `Tasks/Rundowns/` and `Tasks/Pending/` are shielded from the Task folder template
  (a `Noop.md` Templater mapping plus guards inside `Templates/Task.md`); keep it
  that way, both are created fully-formed.
- Idempotent: running it again just rewrites today's note.

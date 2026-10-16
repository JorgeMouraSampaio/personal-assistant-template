---
name: obsidian-book-tasks
description: >-
  Book open tasks from this Obsidian vault's Tasks base into calendar time
  slots, sized by each task's Effort, preferring the daily "Work Block"
  (10:00–12:00) and falling back to any other free gap in the working day,
  without overlapping existing meetings. Use whenever the user
  says "book my tasks", "book the tasks into my calendar", "schedule my tasks",
  "fill my Work Blocks", "book my mornings", or similar. Reads every eligible
  task (Effort set, Booked ? = N), orders them by due date then priority, packs
  them forward into the soonest free time, previews the plan for
  confirmation, then creates the events and flips each task's Booked ? to Y.
  Booking is ALWAYS on demand: tasks are never booked at creation time, only
  when the user asks. Read obsidian-base-cleanup for the vault model first.
---

# Book Tasks (Tasks base → calendar time slots)

Turn effort-estimated tasks into concrete calendar slots. This skill only ever
runs **on demand** (the user explicitly asks to book). It never books at task
creation time. It reads the vault, plans a schedule, **previews it and waits for
confirmation**, then writes calendar events and flips `Booked ?` to `Y`.

## Fixed configuration

Set the first two for yourself before using this skill; the rest are the defaults
this vault ships with.

- **Calendar:** the user's primary calendar, `you@example.com`. Requires a
  calendar tool with `list_events` and `create_event` (any MCP calendar server).
- **Timezone:** `Europe/Lisbon` (use your own; put it on every event, never book
  in UTC).
- **Preferred window:** the daily **`Work Block`** event ∩ `10:00–12:00`. Tasks are
  booked *on top of* the Work Block.
- **Fallback window:** the rest of the working day, `09:00–18:00`. Used only when
  the Work Block cannot take the task. A day with no Work Block is still bookable
  through the fallback.
- **Event title:** exactly `Task #<Task Number>` (e.g. `Task #90`). The task name
  never goes in the title.
- **Event description:** always filled, so the user can read what the slot is for
  from the calendar alone. Plain text, this shape:

  ```
  <Task title>
  Due: YYYY-MM-DD (Ddd) · <container> · Priority <High/Medium/Low>
  Effort: <n> min

  <Notes, with [[wikilinks]] flattened to plain names>

  Links: <the task's Links URLs, one per line>
  Open in Obsidian: obsidian://open?vault=personal-assistant&file=Tasks%2FTask%20Notes%2F<URL-encoded title>
  ```

  Drop any line whose value is blank (no due date, no priority, no Notes, no
  Links). The container is the `Project`, else the `Initiative`, same as the
  rundown. Keep the Notes text as written; do not summarise or add to it.
  No location, no attendees.

## What counts (task selection)

Scan every `Tasks/Task Notes/*.md` tagged `Task`. A task is **eligible** to book
when ALL hold:

- `Status` is `Not Started` or `In Progress` (never `Blocked`, `Completed` or
  `Discarded`: a Blocked task waits on someone else, so a slot would sit idle).
- `Effort` is set to a positive number of minutes (a multiple of 15). This is the
  slot length. **Blank Effort ⇒ not bookable**: collect these and report them at
  the end as "needs an Effort estimate", never guess one.
- `Booked ?` is `N`. Tasks already `Y` are skipped (this is what makes re-runs
  safe and idempotent).

## Ordering

Sort the eligible tasks:

1. **Due date ascending** (soonest first). **Undated tasks sort last.**
2. Then **priority**: `High` → `Medium` → `Low` → blank.
3. Then **Task Number ascending** (stable tiebreak).

Dated tasks are therefore always placed before undated backlog tasks.

## The bookable windows each day

Each business day has **two tiers** of bookable time. Tier A is preferred; tier B
is the overflow. Both are computed from the same event list:

1. List events on the primary calendar for that day (`list_events`, that day's
   bounds, `timeZone: Europe/Lisbon`).
2. **Blockers = every busy event that overlaps the day**: real meetings AND small
   events like `Emails`, `Lunch`, `drop off the kid`, `Closing tasks`,
   `Manager Time`, syncs, reminders, `Out of office`, AND any existing `Task #NN`
   events already booked. The **`Work Block` is the one exception** and is NOT a
   blocker (tasks sit on top of it).
3. **Tier A (preferred) = (Work Block start–end) ∩ (10:00–12:00), minus blockers.**
   The Work Block is sometimes shorter or longer than two hours (e.g. 10:00–11:30,
   10:00–13:00); always intersect with the real event, never assume a full
   10:00–12:00. **No Work Block that day ⇒ tier A is empty**, and the day is still
   bookable through tier B.
4. **Tier B (fallback) = (09:00–18:00) minus tier A's span, minus blockers.** These
   are the ordinary gaps between meetings: a free late morning, an open early
   afternoon, a gap before the last call of the day.
5. **Free intervals** in each tier are the contiguous gaps a task can go into.

## Placement (fill forward, urgent first)

Walk the ordered task list. For each task, scan days from the earliest available
one forward and drop it into the **first free interval that is ≥ its Effort**.
**Within each day, exhaust tier A before looking at tier B**, then move to the
next day:

```
for each task (urgent first):
  for each day from the cursor forward:
    if a tier A gap fits  → place it there, done
    if a tier B gap fits  → place it there, done
  → otherwise: no capacity in horizon
```

This keeps the Work Block as the default home for task work while letting a task
that does not fit there land the same day instead of slipping a week. Prefer the
**earliest** fitting gap within the chosen tier.

- Start the day-cursor at **now**: a day is in play if any of its windows has not
  fully passed. Never book a slot that starts in the past.
- A task occupies a **single contiguous gap** (no splitting a task across two gaps
  or across days). Place it at the gap's start; shrink that gap by the Effort so
  the next task can follow immediately after.
- Because the list is ordered urgent-first, the most pressing tasks claim the
  soonest slots; the rest spill to later days.
- **Booked-late:** if a task's chosen slot starts **after its due date**, still
  book it (earliest later slot) and mark it `⚠ late` in the preview and summary.
- **Horizon:** iterate forward up to ~15 business days. Anything still unplaced
  after that is reported as "no capacity in horizon", not booked.

## Preview, then confirm (required)

Never create events before the user confirms. Present the full plan as a table,
grouped by day, e.g.:

```
Tue 2026-07-21  (Work Block 10:00–12:00, free 10:15–12:00)
  10:15–11:00  Task #90  Review the draft proposal               (due 07-21, High)
  11:00–11:20  Task #53  Send the vendor quote to Sarah           (due 07-21)
  11:20–11:35  Task #79  Sync with Alex on template versioning    (due 07-21)
  14:30–15:30  Task #80  Draft the Q3 pitch deck ...              (due 07-22) [outside Work Block]
Wed 2026-07-22  (Work Block 10:00–12:00, free 10:30–12:00)
  10:30–10:40  Task #88  Confirm Ines + Pedro ready ...          (due 07-21) ⚠ late
  ...
Not booked (needs Effort): #20, #66
Not booked (no capacity in horizon): none
```

Mark every tier B slot `[outside Work Block]` so the user can see at a glance
which bookings left the Work Block, and why (the Work Block was full or absent).

Show the task name in the **preview** (so the user can sanity-check). It goes in
the event description, never in the title. On explicit "go"/"yes"/"book it", proceed.
If the user edits the plan ("move #90 to Wednesday", "skip #74"), re-plan and
re-preview.

## Writing the bookings

For each planned slot, in order:

1. `create_event` on `you@example.com`: `summary: "Task #<n>"`,
   `startTime`/`endTime` as ISO 8601 with `timeZone: Europe/Lisbon`,
   `description` built as in Fixed configuration, no attendees.
2. Set that task note's `Booked ?` frontmatter to `Y` (exact-match the `Booked ?:`
   line; change nothing else, never touch `Status`, `Effort`, `Due Date`, etc.).

Then post a summary: what was booked (day, time, `Task #`), what was booked late,
what could not be booked and why (missing Effort / no capacity).

## Idempotency & re-runs

Re-running is safe: `Booked ? = Y` tasks are skipped, and their existing
`Task #NN` events are treated as blockers, so a second run only slots newly
eligible tasks into whatever free time remains. There is no separate booking-log
file; `Booked ?` on each note plus the `Task #NN` events on the calendar ARE the
state.

## Unbooking (when asked)

To unbook a task ("unbook #90", "clear my Task bookings"): delete its `Task #<n>`
calendar event(s) (`search_events`/`list_events` for the title, then
`delete_event`) and set the task's `Booked ?` back to `N`. Do not touch anything
else.

## Guardrails

- **On demand only.** Never book as a side effect of creating a task or any other
  skill. Only when the user asks in this session.
- **Confirm before writing.** Always preview and get an explicit go before the
  first `create_event`. Calendar writes are bulk and outward-facing.
- **Never** book outside `09:00–18:00`, over a meeting or any other blocker, or in
  a slot that starts in the past. Booking outside the Work Block is allowed and
  expected once the Work Block is full or absent; booking over a **blocker** never
  is, unless the user explicitly asks for that one slot.
- The title is always `Task #<n>` and nothing else. The description always
  carries the readable task detail. This holds for one-off bookings of a single
  task too, not only bulk "book my tasks" runs.
- **Never** split a task across gaps/days, and never resize a task to fit (Effort
  is fixed; if it does not fit anywhere in the horizon, leave it unbooked).
- Task notes: only ever flip `Booked ?`. This skill is otherwise read-only with
  respect to task frontmatter.
- These events have **no attendees**, so no invitations are sent. If the user ever
  asks to book a task slot that includes other people, that becomes an invite and
  needs its own explicit confirmation.

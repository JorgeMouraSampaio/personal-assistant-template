---
name: obsidian-process-meeting
description: >-
  Process a raw Granola/meeting note from this Obsidian vault's Meetings/ inbox
  into a clean, crosslinked meeting note (in Meetings/Processed Meetings/) plus
  actionable tasks. Use whenever the user says "process this meeting", "process my
  Granola notes", "turn this transcript into tasks", "what are my action items /
  CTAs from this meeting", or points at a note in Meetings/. Files a crosslinked note
  into Meetings/Processed Meetings/ (Date, Attendees, Company, Project as wikilinks),
  files the raw transcript into Meetings/Transcripts/ tagged RawTranscript, auto-creates any missing
  People/Company/Project notes, extracts every action item / call-to-action, and
  turns the USER'S OWN action items into Task notes following
  obsidian-create-task. This is task-entry METHOD 2; METHOD 1 is obsidian-create-task.
---

# Process Meeting (Granola inbox → filed note + transcript + tasks)

Granola dumps raw meeting notes and transcripts into `Meetings/Inbox/` as
Markdown. Those raw files have useful content but no vault structure: no
`[[wikilinks]]`, Granola's own frontmatter, and action items as loose prose. This
skill turns one raw meeting into (a) a crosslinked meeting note filed in
`Meetings/Processed Meetings/`, (b) its raw transcript filed in
`Meetings/Transcripts/`, and (c) real Tasks for the user's own action items.

**Granola's summary note is the canonical record — never write your own.**
Processing = relabel and file Granola's note, crosslink it, and mine **it** (not
the transcript) for tasks. Claude adds frontmatter, wikilinks, and an
`## Action items` section — nothing else.

**Wait for the summary note.** Granola generates its AI summary minutes-to-hours
after the meeting; the transcript often syncs first. A transcript alone
(`type: transcript`, no paired `type: note` file with the same `granola_id`) is
**not processable** — leave both files in the inbox untouched and skip the
meeting; a later sync/scan will pick it up once the summary lands. Never
reconstruct a summary from the transcript.

Read `obsidian-base-cleanup` (vault model, wikilink quoting, `People/<Company>/`
folders, entity creation) and `obsidian-create-task` (task schema, numbering,
Status handling) first — this skill composes both.

## Folder model — why we move things OUT of the inbox, and rename the id key

Everything meeting-related lives under `Meetings/`. The **Granola Sync** plugin
dumps raw notes and raw transcripts into `Meetings/Inbox/` (its `customBaseFolder`
and `customTranscriptBaseFolder` both point there) — a **quarantined** inbox, so
the plugin's re-sync churn stays out of the `Meetings/` root. Two sibling
subfolders are permanent, Claude-owned homes:

| Folder | Owner | Contents |
|---|---|---|
| `Meetings/Inbox/` | Granola sync | raw notes **and** raw transcripts — **inbox**, transient, re-dumped every sync (plus a `README.md` explaining it) |
| `Meetings/Processed Meetings/` | this skill | clean crosslinked meeting notes — **the Meetings base** |
| `Meetings/Transcripts/` | this skill | raw transcripts, tagged `RawTranscript`, kept forever |

Processing **moves** artifacts from `Meetings/Inbox/` into the two permanent
subfolders. The Meetings base is folder-filtered on
`file.inFolder("Meetings/Processed Meetings")`; `Meetings/Inbox/` is excluded from
Templater folder-templates so the sync never triggers a template.

**Why quarantine the inbox?** Granola keeps re-fetching any meeting still in its
sync window and re-dumps the raw copy on every sync (~every 30 min), so deleting a
raw file from the inbox is whack-a-mole — it comes back. Confining that churn to
`Meetings/Inbox/` keeps the `Meetings/` root tidy (just the two permanent folders
+ `Meetings.base`), and the Meetings base never shows a duplicate because it only
indexes `Processed Meetings/`.

### Why we rename `granola_id` → `granola_ref` (critical)

The plugin builds a **vault-wide** cache of every Markdown file that carries a
`granola_id` frontmatter key (`getMarkdownFiles()` → keyed by `granola_id`). On
each sync it looks up that id **anywhere in the vault** and relocates/overwrites
that file back into `Meetings/Inbox/` with fresh Granola content.
**Moving a file out of the inbox is not enough** — as long as it still carries
`granola_id`, the plugin owns it and will drag it back and clobber it, no matter
what folder it sits in.

The fix: on every filed copy (processed note **and** filed transcript), rename the
frontmatter key `granola_id` → `granola_ref`. The plugin's cache never indexes
`granola_ref`, so the filed copy becomes invisible to the sync and is permanently
safe. Keep the value unchanged — it is still our dedup key.

**Dedup key = `granola_ref`.** Before processing a raw note, look for a note in
`Meetings/Processed Meetings/` whose `granola_ref` equals the raw note's
`granola_id`. If one exists it is already done — delete the re-synced raw note
(and its raw transcript) from `Meetings/Inbox/` and skip. This is what makes
re-syncs idempotent: the plugin re-dumps the raw note (with `granola_id`) into
`Meetings/Inbox/`, and we recognize it as already-filed via the matching
`granola_ref`.

**Dates.** The source of truth is **when the meeting actually happened** — the
date portion of Granola's `created` field, NOT `updated` (which is just the last
edit). The processed note and its transcript must carry the **same `Date`**.

## Workflow

### 1. Read the raw meeting file — and gate on the summary note
Read the target note in `Meetings/Inbox/` (the user names it, or process the most
recently modified; the auto-scan does all not-yet-filed ones).
**Gate:** the processable unit is Granola's **summary note** (`type: note`). If
only the transcript exists for a meeting, stop — report "summary not yet
generated, will process on a later pass" and leave the inbox untouched. Note the
summary's `granola_id`, `created` (→ meeting date), and its paired transcript at
`Meetings/Inbox/<title>-transcript.md` (same inbox). Skip already-done ones via the
`granola_ref` dedup check above. Identify: **date**, **attendees** (names + org),
the **company/client**, the **project** if any, **URLs/docs**, and **action items**.

### 2. Resolve & crosslink entities (auto-create missing)
For each attendee, company, and project:
- If a note already exists (match by basename), link it.
- If not, **create it** per `obsidian-base-cleanup`: people under
  `People/<Company>/` (colleagues at your own company → `People/<YourCompany>/`; external → their
  own company, creating a `Company`-tagged note if needed; prospective hires or
  unaffiliated contacts → `Company: External`, `People/External/`), projects under
  `Projects/<Company>/`. Fill only what the meeting honestly tells you; leave the
  rest blank.
- A person's `Company` is their employer, inferred from email domain or context
  (`@amazon.com` → AWS, `@yourco.com` → your own company). "Sales team" and other group labels
  are not people — don't link or create them.

**Also crosslink people named in the body, not just attendees.** Granola's
summaries mention colleagues and contacts in prose (e.g. "Julian will…", "per
Helena"). Scan the summary for every **person** named beyond the attendees and,
for each one you can confidently identify, resolve or **create** their note the
same way (matching an existing note by basename first — mind transcription
variants like Rulian/Rullian). This is how new people enter the vault. They get
linked via the `## People mentioned` section (step 3), **not** by editing links
into Granola's verbatim body. Judgment: create a note for any clearly-real,
placeable person (especially internal colleagues); **skip** garbled/uncertain
names and fleeting one-off mentions you can't place. New *companies/projects*
named in passing are only worth creating when they're clearly a real, relevant
entity — otherwise leave them as prose.

### 3. File Granola's summary note into `Meetings/Processed Meetings/`
Create `Meetings/Processed Meetings/<YYYYMMDD> <title>.md` — **date-prefix the
filename** with the meeting date (Granola's `created`, digits only, e.g.
`20260521 GTM session.md`) so the Meetings base sorts and reads chronologically.
The wikilink target is therefore the prefixed basename (`[[20260521 GTM session]]`),
and the paired transcript's `Meeting:` back-link must use it. Give it clean,
vault-shaped frontmatter, then **Granola's note body verbatim** (this is a
move-and-relabel, never a rewrite — Granola's summary IS the meeting record), then
two appended sections: `## Action items` and `## People mentioned`:

```yaml
---
tags:
  - Meeting
Date: 2026-05-21            # date the meeting happened (from Granola `created`)
Attendees:
  - "[[Jane Doe]]"
  - "[[John Smith]]"
Company: "[[AWS]]"          # or blank
Project: "[[Widget]]"       # or plain blank; never wikilink "General task"
Links:                      # any doc/ticket URLs mentioned
  - https://notes.granola.ai/t/...
Transcript: "[[<title>-transcript]]"
granola_ref: 912d64fe-...    # renamed from granola_id so the sync can't reclaim this note; our dedup key. Drop Granola's title/type/created/updated
---
```

Do **not** carry over an `updated`/`Updated` field — it is intentionally not part
of the Meetings base.

### 4. File the transcript into `Meetings/Transcripts/`
Move the paired transcript from `Meetings/Inbox/` to
`Meetings/Transcripts/<title>-transcript.md` (transcripts are **not** date-prefixed
— only the processed note in the Meetings base is), keeping its body verbatim, with
frontmatter: `tags: [RawTranscript]`, `Date:` = the same meeting date as the note,
`Meeting: "[[<YYYYMMDD> <title>]]"` back-link (the date-prefixed note basename), and
its `granola_id` **renamed to `granola_ref`** (same reason as the note — otherwise
the sync reclaims it). The move deletes it from `Meetings/Inbox/`. The note's
`Transcript` field points back at this file: `Transcript: "[[<title>-transcript]]"`.

### 5. Extract action items / CTAs — from the summary note ONLY
Scan **Granola's summary note** (especially its "Next Steps" / action bullets)
for every action item, follow-up, commitment, or CTA. Capture the **action**,
**owner**, any **due date**, and related **people/links**. List them **all** under
`## Action items` in the processed note so it stays a complete record.
The transcript is stored for reference only — **never mine it for tasks**
(auto-transcription is too noisy to be a reliable source of commitments; Granola's
summary is the curated record of what was actually agreed).

### 6. Turn the USER'S OWN action items into Tasks
Only items **the user** owns become Tasks ("I'll…", "<user> will…", assigned to
the user). If the user was not even an attendee, none are theirs — create no tasks
and say so. For each of the user's items, create a Task per `obsidian-create-task`:
- Next free `Task Number` (scan `Tasks/**`), `Status: Not Started`.
- `Project`/`Company` inherited from the meeting (plain `General task` if none).
- `Stakeholders`: the other people tied to that item.
- `Due Date`: resolve relative dates against the meeting date / today; else blank.
- `Links`: any URL; and `Notes: From [[<processed meeting basename>]]` (the
  processed note's basename is date-prefixed — see step 3).
- File flat into `Tasks/Task Notes/` (no per-project subfolders). No kanban step —
  the boards are TaskNotes views over the `Status` property and pick the task up
  automatically.

In `## Action items`, render the user's items as task links (`- [ ] [[Task title]]`)
and others' as plain bullets naming the owner.

### 7. Write the `## People mentioned` section
After `## Action items`, append `## People mentioned` — a bullet list of `[[Person]]`
wikilinks for everyone **named in the body who is not already an attendee** (attendees
are already linked in frontmatter, so don't repeat them). This backlinks the meeting
onto each person's note without touching Granola's verbatim body — it's the payoff of
the step-2 crosslinking. One bullet per person; add a few words of context where it
helps (`- [[Rulian]] — keen to take the bio proposal-review role`). If nobody beyond
the attendees is named, write `- None.`

### 8. Clean the inbox & verify
Delete the raw note from `Meetings/Inbox/` (its content now lives in
`Meetings/Processed Meetings/`; this avoids a basename collision and keeps the inbox
to only unprocessed items). Run the broken-wikilink check from
`obsidian-base-cleanup`: note→attendees, note→company/project, note↔transcript,
note→mentioned-people, and task↔note links must all resolve. Then summarize:
attendees linked, people/entities created, transcript filed, and which action items
became tasks (numbers) vs. left as context.

## Automatic processing (the auto-scan)
A local scheduled task (`obsidian-process-meetings` in `~/.claude/scheduled-tasks/`)
runs periodically and processes new meetings unprompted. It scans `Meetings/Inbox/`
(`Meetings/Inbox/*.md`, skipping `README.md`), dedups against
`Meetings/Processed Meetings/` by matching `granola_ref` to each raw note's
`granola_id`, and runs this workflow on anything not yet filed — moving finished
notes/transcripts out of the inbox so the next Granola sync can't overwrite them.
Transcript-only meetings (no `type: note` file yet) are left alone; they get
picked up on a later pass once Granola's summary has synced.

## Judgment & safety
- **Never rewrite a note or transcript body.** Copy it verbatim; only add
  vault-shaped frontmatter + an `## Action items` section. Never substitute your
  own summary for Granola's — if the summary hasn't synced yet, wait.
- **Move, don't leave copies in the inbox** — a leftover raw note in
  `Meetings/Inbox/` with the same basename as the filed one breaks wikilink
  resolution.
- Distinguish a real **CTA/action item** from mere discussion — don't manufacture
  tasks from every sentence.
- Prefer a blank property over an invented value; don't guess a due date or owner.
- Task `Status` frontmatter is the single source of truth (new tasks start
  `Not Started`); the kanban boards are TaskNotes views and update themselves.

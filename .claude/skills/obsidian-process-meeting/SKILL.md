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
  PROPOSES the USER'S OWN action items as tasks for validation. the user signs off
  before any Task note is written (unattended runs stage them in
  Tasks/Pending/Proposed Tasks.md). This is task-entry METHOD 2; METHOD 1 is
  obsidian-create-task.
---

# Process Meeting (Granola inbox → filed note + transcript + tasks)

Granola dumps raw meeting notes and transcripts into `Meetings/Inbox/` as
Markdown. Those raw files have useful content but no vault structure: no
`[[wikilinks]]`, Granola's own frontmatter, and action items as loose prose. This
skill turns one raw meeting into (a) a crosslinked meeting note filed in
`Meetings/Processed Meetings/`, (b) its raw transcript filed in
`Meetings/Transcripts/`, and (c) a **validated** set of Tasks for the user's own
action items.

**Tasks are proposed, never auto-created.** Filing the meeting (note, transcript,
entities, `## Action items`) happens straight away and needs no sign-off: the
record has to get out of the inbox before the next Granola sync. Writing Task
notes is gated: the user validates the extraction first. See
"The validation gate" below; it is the one hard rule of this skill.

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
| `Meetings/Processed Meetings/` | this skill | clean crosslinked meeting notes, bucketed into `<YYYY>-wk.<WW>/` week subfolders — **the Meetings base** |
| `Meetings/Transcripts/` | this skill | raw transcripts, tagged `RawTranscript`, bucketed into matching `<YYYY>-wk.<WW>/` week subfolders, kept forever |

Processing **moves** artifacts from `Meetings/Inbox/` into the two permanent
subfolders, and within each into an **ISO-week subfolder** named
`<YYYY>-wk.<WW>` (e.g. `2026-wk.29`, the week the meeting happened). The Meetings
base is folder-filtered on `file.inFolder("Meetings/Processed Meetings")` — a
recursive path match, so the week subfolders are still indexed. `Meetings/Inbox/`
is excluded from Templater folder-templates so the sync never triggers a template.

**Computing the week bucket.** From the meeting date (`YYYY-MM-DD`), the bucket is
`<YYYY>-wk.<WW>` where `<WW>` is the zero-padded ISO week number and `<YYYY>` is
the note's calendar year. Compute it, e.g.
`date -j -f "%Y-%m-%d" "2026-07-16" "+%V"` → `29` → `2026-wk.29`. Create the
subfolder if it doesn't exist. Both the processed note and its transcript go into
the **same** week bucket (they share the same `Date`).

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

**Dedup key = `granola_ref`.** Before processing a raw note, look for a note
**anywhere under** `Meetings/Processed Meetings/` (search its week subfolders
recursively) whose `granola_ref` equals the raw note's `granola_id`. If one exists it is already done — delete the re-synced raw note
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
**Include the user as an attendee whenever he participated.** Granola records from
the user's account and routinely omits the account owner from the attendee list — he
shows up as **"You"** in the transcript. So by default add `- "[[Jane Doe]]"`
to the processed note's `Attendees` (listed **first**), even when Granola's list
doesn't name him. **Exception:** some notes are imports/shares of meetings the user did
**not** attend — the tell is a transcript with **no "You" segments** (all speakers
are named/"Guest"). In that case do **not** add him; instead note in the body that
the user was not an attendee (so his own action-item weighting in step 5 is skipped).

For each attendee, company, and project:
- If a note already exists (match by basename), link it.
- If not, **create it** per `obsidian-base-cleanup`: people under
  `People/<Company>/` (your own colleagues → `People/<your company>/`; external → their
  own company, creating a `Company`-tagged note if needed; prospective hires or
  unaffiliated contacts → `Company: External`, `People/External/`), projects under
  `Projects/<Company>/`. Fill only what the meeting honestly tells you; leave the
  rest blank.
- A person's `Company` is their employer, inferred from email domain or context
  (`@amazon.com` → AWS, your own domain → your own company). "Sales team" and other group labels
  are not people — don't link or create them.

**Also crosslink people named in the body, not just attendees.** Granola's
summaries mention colleagues and contacts in prose (e.g. "Alex will…", "per
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

### 3. File Granola's summary note into `Meetings/Processed Meetings/<YYYY>-wk.<WW>/`
Create `Meetings/Processed Meetings/<YYYY>-wk.<WW>/<YYYYMMDD> <title>.md` — file it
into the meeting's **ISO-week subfolder** (see "Computing the week bucket" above;
create it if missing) and **date-prefix the filename** with the meeting date
(Granola's `created`, digits only, e.g. `2026-wk.21/20260521 GTM session.md`) so the
Meetings base sorts and reads chronologically. The wikilink target is folder-
independent — still the prefixed basename (`[[20260521 GTM session]]`),
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
granola_ref: 0f1e2d3c-...    # renamed from granola_id so the sync can't reclaim this note; our dedup key. Drop Granola's title/type/created/updated
---
```

Do **not** carry over an `updated`/`Updated` field — it is intentionally not part
of the Meetings base.

### 4. File the transcript into `Meetings/Transcripts/<YYYY>-wk.<WW>/`
Move the paired transcript from `Meetings/Inbox/` to
`Meetings/Transcripts/<YYYY>-wk.<WW>/<title>-transcript.md` — the **same** week
subfolder as its processed note (create it if missing). Transcript filenames are
**not** date-prefixed (only the processed note in the Meetings base is); the week
subfolder gives them the chronological grouping. Keep its body verbatim, with
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

### 6. Propose the USER'S OWN action items as tasks (do NOT create them yet)
Only items **the user** owns are task candidates ("I'll…", "the user will…", assigned to
the user). the user is always an attendee (step 2), so always weigh the action items for
his own. **"Note-taker" (or "notetaker") in a Granola summary means the user** — he owns
the recording account — so any action assigned to the note-taker is his. For each of
the user's items, work out the full task shape per `obsidian-create-task`
(title, `Project`/`Company` inherited from the meeting with plain `General task` if
none, `Stakeholders` = the other people on the item, `Due Date` resolved against the
meeting date / today or blank, `Links`, and
`Notes: From [[<processed meeting basename>]]`), then **stop**. Nothing is written to
`Tasks/Task Notes/` until step 6b.

In `## Action items`, render the user's candidates as **plain** unchecked bullets
suffixed `(proposed task, pending validation)`, never as `[[wikilinks]]`, since the
task note doesn't exist yet and would be a broken link. Others' items stay plain
bullets naming the owner.

### 6b. Validate, then create
Run the validation gate below. Only the items the user approves become Task notes:
- Assign the **next free `Task Number`** at creation time (re-scan `Tasks/**`;
  never reuse a number projected earlier if the vault moved on), `Status: Not Started`.
- File flat into `Tasks/Task Notes/` (no per-project subfolders). No kanban step —
  the boards are TaskNotes views over the `Status` property and pick the task up
  automatically.
- Back-fill `## Action items` in the processed note: each created task's bullet
  becomes `- [ ] [[Task title]]`. A rejected proposal keeps a plain bullet with
  `(not tracked as a task)`; the action item stays in the record either way.

If the gate can't be resolved on this run (unattended scan, or the user doesn't answer),
the proposals go to the staging queue and the meeting stays fully filed with its
`(proposed task, pending validation)` bullets. That is a complete, valid end state.

### 7. Write the `## People mentioned` section
After `## Action items`, append `## People mentioned` — a bullet list of `[[Person]]`
wikilinks for everyone **named in the body who is not already an attendee** (attendees
are already linked in frontmatter, so don't repeat them). This backlinks the meeting
onto each person's note. (Correcting an obvious ASR name typo in the body — per the
Judgment & safety exception — is the only editing allowed; otherwise the body stays
verbatim.) It's the payoff of the step-2 crosslinking. One bullet per person; add a few words of context where it
helps (`- [[Alex Reed]] — keen to take the proposal-review role`). If nobody beyond
the attendees is named, write `- None.`

### 8. Clean the inbox & verify
Delete the raw note from `Meetings/Inbox/` (its content now lives in
`Meetings/Processed Meetings/`; this avoids a basename collision and keeps the inbox
to only unprocessed items). Run the broken-wikilink check from
`obsidian-base-cleanup`: note→attendees, note→company/project, note↔transcript,
note→mentioned-people, and task↔note links must all resolve. Pending proposals are
plain text, not wikilinks, so they can't break the check; if you see a
`- [ ] [[Task title]]` bullet whose note doesn't exist, a task was approved but never
written. Then summarize: attendees linked, people/entities created, transcript filed,
and which action items were **proposed** as tasks (awaiting validation, or staged) vs.
left as context. Report created tasks with their real numbers only once they exist.

## The validation gate

Meeting-derived tasks are **proposals until the user says otherwise**. Extraction from a
summary is a judgment call (an owner can be misread, a "we should maybe" can look like
a commitment, one item can split into three), so the user reviews the list before it
lands in the Tasks base. This gate applies to **METHOD 2 only**: tasks the user asks for
directly (`obsidian-create-task`, METHOD 1) are already validated by the asking.

### Presenting the proposals (interactive runs)

After filing the meeting(s), report the extraction in this exact shape, one block per
meeting, blocks separated by a blank line:

```
I extracted these tasks from [[20260716 GTM session]]:

* #412 - Send the migration outreach to Sarah Chen (AWS, due 2026-08-04)
* #413 - Draft the migration offering one-pager (Migration Offering)

I extracted these tasks from [[20260717 Acme sync]]:

* #414 - Review the benchmark report

Approve all, or tell me which to drop or edit.
```

- `#XXX` is the **projected** `Task Number`: current highest across `Tasks/**` plus
  one, incrementing down the list, continuing across meetings in the batch. It is a
  handle for the conversation, not a reservation. Re-derive the real number when you
  actually create the note, and if it shifted, report the number you assigned.
- The bullet is `* #XXX - <Task title>`. Append only the metadata that exists, in
  parentheses, comma-separated: company or project, `due YYYY-MM-DD`, stakeholders if
  they add signal. Blank fields are simply absent. No metadata at all is fine.
- One bullet per task, one block per meeting, even for a single task or a single
  meeting. A meeting that yielded none gets no block; say so in one line
  (`No tasks for me out of [[20260718 standup]].`).
- Then **wait**. Do not create anything in the same turn as the proposal.

### Reading the answer

- Blanket yes ("approve", "all good", "go ahead", "yes") → create every proposal.
- Selective ("drop 2", "not #413", "just the first one") → create the rest, delete
  the dropped ones.
- Edits ("#412 is due Friday", "retitle 413 to X", "that one is General task") →
  apply, then create. Re-show the corrected list only if the edits were substantial.
- Silence or a topic change → nothing is created; stage the proposals (below).
- **Never infer approval** from a message that only discusses the meeting, and never
  treat approval of one meeting's block as covering another's.

### Staging (unattended runs)

An unattended run (the auto-scan, or any run with nobody to ask) files the meeting
normally, then appends one block per meeting to `Tasks/Pending/Proposed Tasks.md`:

```markdown
### From [[20260716 GTM session]] (staged 2026-07-29)
- **Send the migration outreach to Sarah Chen**
  - Project: [[Migration Offering]] · Company: [[AWS]]
  - Stakeholders: [[Sarah Chen]], [[John Smith]]
  - Due Date: 2026-08-04
  - Links:
  - Notes: From [[20260716 GTM session]]
```

Store no `Task Number` (it would go stale). Omit or leave blank any field the meeting
didn't give you. That note is **untagged** so it stays out of the Tasks base, the
kanban boards, and the rundown; keep it that way. The daily rundown flags a non-empty
queue, and any later interactive run reads the queue back in the format above (mixing
freshly-extracted blocks and staged ones, projected numbers computed at read-back).
On approval, create the task and **delete its block** from the queue; on rejection,
delete the block and mark the meeting's action-item bullet `(not tracked as a task)`.
An empty queue means the file ends after its header comment.

## Automatic processing (the auto-scan)
A local scheduled task (`obsidian-process-meetings` in `~/.claude/scheduled-tasks/`)
runs periodically and processes new meetings unprompted. It scans `Meetings/Inbox/`
(`Meetings/Inbox/*.md`, skipping `README.md`), dedups against
`Meetings/Processed Meetings/` (recursively, across week subfolders) by matching
`granola_ref` to each raw note's `granola_id`, and runs this workflow on anything
not yet filed — moving finished notes/transcripts out of the inbox and into their
`<YYYY>-wk.<WW>/` buckets so the next Granola sync can't overwrite them.
Transcript-only meetings (no `type: note` file yet) are left alone; they get
picked up on a later pass once Granola's summary has synced.
It **never creates Task notes** (nobody is there to validate): it stages its task
proposals in `Tasks/Pending/Proposed Tasks.md` per the validation gate above.

## Judgment & safety
- **Never rewrite a note or transcript body.** Copy it verbatim; only add
  vault-shaped frontmatter + an `## Action items` section. Never substitute your
  own summary for Granola's — if the summary hasn't synced yet, wait.
  - **One exception — obvious ASR typos in proper nouns.** You may correct clear
    transcription manglings of names — people, companies, projects, events,
    products — in the summary body to their known-correct spelling (e.g.
    a mis-heard company name, a colleague's name spelled phonetically, an
    acronym heard wrong). Only when
    the correct entity is unambiguous; never rephrase content, fix "regular" prose,
    or change meaning. If unsure, leave it and flag it. The transcript stays fully
    verbatim — correct names only in the summary body.
- **Move, don't leave copies in the inbox** — a leftover raw note in
  `Meetings/Inbox/` with the same basename as the filed one breaks wikilink
  resolution.
- Distinguish a real **CTA/action item** from mere discussion — don't manufacture
  tasks from every sentence.
- **Never write a Task note from a meeting without the user's explicit sign-off.** Filing
  the meeting is unconditional; creating tasks is not. When in doubt, propose or stage.
  Over-proposing is cheap (he drops a line); a task he never agreed to pollutes the
  base, the kanban, and the rundown.
- Prefer a blank property over an invented value; don't guess a due date or owner.
- Task `Status` frontmatter is the single source of truth (new tasks start
  `Not Started`); the kanban boards are TaskNotes views and update themselves.

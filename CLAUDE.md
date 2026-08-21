# Personal Assistant Vault — guide for Claude

A lightweight CRM + project/task system built on Obsidian **Bases** (table views
over notes), **Templater** (note scaffolding + auto-filing), and the **TaskNotes**
plugin (drag-drop kanban views over task notes). Follow the conventions below.
The skills carry the full procedures — read the relevant one before non-trivial
work:

- `.claude/skills/obsidian-base-cleanup/SKILL.md` — entities, crosslinking,
  folders, divisions, Templater, Meetings/Granola.
- `.claude/skills/obsidian-create-task/SKILL.md` — natural-language → tasks
  (task-entry METHOD 1).
- `.claude/skills/obsidian-process-meeting/SKILL.md` — a Granola/meeting note in
  `Meetings/` → crosslinked note + task proposals the user validates before they become
  tasks (task-entry METHOD 2).
- `.claude/skills/obsidian-log-session/SKILL.md` — user finishes a work session →
  session note in `Sessions/<Project>/` (what was done, decisions, task statuses).
- `.claude/skills/obsidian-daily-rundown/SKILL.md` — daily digest of open tasks
  by due date → `Tasks/Rundowns/YYYY-MM-DD Rundown.md` (a scheduled task runs it
  every day at 9am; also on demand: "what's on my plate today").
- `.claude/skills/obsidian-book-tasks/SKILL.md` — book eligible tasks (Effort set,
  `Booked ? = N`) into calendar slots, sized by Effort, ordered by due date then
  priority. Prefers the daily `Work Block` (10:00–12:00) and falls back to any
  other free gap in the working day (09:00–18:00) when the Work Block is full or
  absent; on demand only ("book my tasks"), previews then flips `Booked ?` to `Y`.
- `.claude/skills/obsidian-process-resource/SKILL.md` — a Web Clipper clipping in
  `raw/` → structured Resource note (Resources base) + LLM-owned `wiki/` pages
  ("process my resources" / "compile"; a scheduled task also auto-scans `raw/`).

## Writing style

All responses and output content (drafted messages, note bodies, summaries, chat
replies) must follow ASD-STE100-style English: plain, easy to read, for the
spoken voice. Concretely, avoid:

- Em dashes (`—`). Use commas, colons, parentheses, or separate sentences
  instead. (En dashes in date/number ranges like `500M/3B` or `Aug 17–Nov 1`
  are fine.)
- Antithesis, contrasting pairs, negative parallelisms, and negative anaphoras
  (e.g. "It's not X, it's Y" repeated or echoed structures built on contrast).
- Corrective negation ("not just X, but Y").
- Parataxis and paragraph pinning (short, disconnected clause-stacking used to
  sound punchy).
- Summary beats, landing sentences, and setup/payoff constructions (no
  "the bottom line is…" closers or telegraphed reveals).
- Rhetorical crutches and performed enthusiasm.
- The rule of three (no forced triads of examples or adjectives).
- Parallel sentence structures repeated within a paragraph.
- Throat-clearing openers ("So,", "Well,", "Great question,").
- Stacked noun phrases (heavy noun-pileups in place of a verb phrase).
- Filler intensifiers (genuinely, really, truly, actually).
- Corporate-register verbs (leverage, underscore, reflect, etc.) and
  nominalization (turning verbs into abstract nouns, e.g. "utilization" for
  "using").
- Hedging qualifiers (somewhat, arguably, to some extent).

Vary sentence length unpredictably rather than settling into a rhythm.

## Bases, folders, membership

| Base | Folder | In the base when… | Key properties |
|---|---|---|---|
| Companies | `Companies/` | tagged `Company` | Division, Stakeholders, Projects, Notes |
| People | `People/<Company>/` | tagged `People` | Company, Division, Role, Email, LinkedIn, Notes |
| Projects | `Projects/<Company>/` | tagged `Project` | Project Number, Company, Start/End Date, Status, Link |
| Tasks | `Tasks/Task Notes/` (flat) | tagged `Task` | Task Number, Status, Project, Stakeholders, Company, Created/Due Date, Priority, Effort, Booked ?, Notes, Links |
| Meetings | `Meetings/Processed Meetings/` | is in the `Meetings/Processed Meetings/` folder | Date, Attendees, Company, Project, Links, Transcript |

Meeting notes are filed with a **`YYYYMMDD ` date prefix** (`20260521 GTM session.md`)
so the Meetings base reads chronologically; the base surfaces a **Transcript**
column linking each note to its raw transcript. Transcripts themselves are not
date-prefixed. Both processed notes and transcripts are then bucketed into
`<YYYY>-wk.<WW>/` ISO-week subfolders (e.g. `2026-wk.29/`); a note and its
transcript always share the same bucket. The base's
`file.inFolder("Meetings/Processed Meetings")` filter is a recursive path match,
so the week subfolders stay indexed.
| Sessions | `Sessions/<Project>/` | tagged `Session` | Date, Project, Company, Tasks Completed, Tasks In Progress |
| Resources | `Resources/` (flat) | tagged `Resource` | Title, Link, Type, Author, Company, Project, Date, Note, Raw |

`raw/` and `wiki/` are **not** bases: `raw/` is the immutable Web Clipper inbox and
`wiki/` pages are deliberately **untagged** (so they never enter a base, kanban, or
the rundown). See the "Wiki & Resources" section below.

Every org — clients, your own company, vendors, consultancies — gets a note
tagged `Company` and appears in the Companies base; their people live in
`People/<Org>/`. People who belong to no tracked org — prospective hires,
candidates, or contacts who aren't your own company and aren't a client — get
`Company: External` (plain text, never a wikilink, like `General task`) and live
in `People/External/`; `External` is a catch-all label, never a real company note
and never in the Companies base.

**Meeting flow (Granola).** Everything meeting-related lives under `Meetings/`. The
Granola sync dumps raw notes **and** raw transcripts into `Meetings/Inbox/` — a
**quarantined** inbox (transient, re-dumped every sync) so the churn stays out of
the `Meetings/` root. The `obsidian-process-meeting` skill **moves** Granola's
summary note into `Meetings/Processed Meetings/` (Granola's summary IS the meeting
record — Claude only adds frontmatter/crosslinks/action items, never writes its own
summary; if only the transcript has synced, wait) and its transcript into
`Meetings/Transcripts/` (tagged `RawTranscript`), then deletes the inbox copies.
Tasks are mined from the summary note, never from the transcript. Beyond the
attendees (linked in frontmatter), the skill crosslinks **everyone named in the
summary body** — creating notes for new people (this is a main way people enter
the vault) and listing them in a `## People mentioned` section. Claude leaves
Granola's verbatim body intact **except** for correcting obvious transcription
(ASR) typos in proper nouns — people, companies, projects, events, products — to
their known-correct spelling (e.g. a mis-heard company name or a colleague's name
spelled phonetically); it never otherwise rewrites, rephrases, or re-summarizes the
body.
Only fix a name when the correct entity is unambiguous; if unsure, leave it and
flag it. On every filed copy it renames the
`granola_id` frontmatter key to **`granola_ref`**: the plugin owns any file carrying
`granola_id` **vault-wide** (it re-indexes and clobbers it wherever it sits), so the
rename — not just the move — is what makes filed notes permanently safe. Never
enrich a note in place while it still has `granola_id`. Dedup matches a raw note's
`granola_id` against filed `granola_ref`; the meeting date is Granola's `created`
(when it happened), never `updated`.

**Meeting tasks are validated before they exist.** Filing a meeting (note,
transcript, entities, `## Action items`) is unconditional and immediate, but Claude
**never writes a Task note off a meeting without the user's explicit sign-off**. It
reports the extraction as `I extracted these tasks from [[meeting]]:` followed by
`* #XXX - Task title` bullets (`#XXX` = the projected Task Number, re-derived for real
at creation), one block per meeting, then waits. Approval can be blanket, selective,
or come with edits; silence creates nothing. Until then the meeting's action-item
bullets read `(proposed task, pending validation)`, as plain text, never wikilinks. The
unattended auto-scan has nobody to ask, so it stages its proposals in
`Tasks/Pending/Proposed Tasks.md` (untagged, so invisible to every base, board, and
rundown) and the daily rundown flags a non-empty queue. The gate is METHOD 2 only: a
task the user asks for directly is validated by the asking.

**Project dashboards & sessions.** Every project note IS its dashboard: `## Overview`,
`## Tasks` (`![[All Tasks.base#Project Board]]` — a TaskNotes drag-drop kanban),
`## Meetings` (`![[Meetings.base#Project Meetings]]`), `## Sessions`
(`![[Sessions.base#Project Sessions]]`), and `## Key Decisions`. The `Project …`
views live in the **master bases** and filter on `file.hasLink(this.file)` (`this`
= the embedding note), so one view definition serves every dashboard — edit it
once, all dashboards change; opened directly in the master base it shows empty
(expected). Nothing is registered anywhere — anything whose frontmatter wikilinks
the project shows up automatically. When the
user finishes a work session, the `obsidian-log-session` skill updates task
Statuses, writes a `Session`-tagged note into `Sessions/<Project>/` (filename
`YYYY-MM-DD <Project> Session`), and appends project-level decisions to the
dashboard's Key Decisions. The **TaskNotes plugin** is remapped to this vault's
schema (tag `Task`, `Status`, `Due Date`, `Project`, `Created Date`) — don't
change its field mapping. Project dashboards also carry a `## Resources`
(`![[Resources.base#Project Resources]]`) section, so any Resource note wikilinking
the project surfaces there too.

**Wiki & Resources (Karpathy LLM-wiki pattern).** A three-layer knowledge system,
additive to and isolated from the CRM/task/meeting flows:

1. **`raw/` (immutable inbox).** The Obsidian **Web Clipper** (browser extension)
   saves external sources here — articles, posts, news, internal docs — as
   `YYYY-MM-DD-<Title>.md` with the clipper's *Clippings* frontmatter
   (`title/source/author/published/created/description/tags:[clippings]`). This is
   the **source of truth**: Claude **reads it but never edits, renames, moves, or
   deletes** a file in `raw/` (unlike the meetings inbox, which is a move-out
   pipeline). Web Clipper setup is browser-side: point its vault + save folder at
   `raw/` (see `raw/README.md`).
2. **`Resources/` (structured, the Resources base).** The `obsidian-process-resource`
   skill turns each clipping into one `Resource`-tagged note with
   `Title, Link, Type (post|article|blogpost|news|internal doc), Author, Company,
   Project, Date, Note (1-sentence summary), Raw` (a back-link to the clipping).
   **Authors are registered as normal People notes and source-orgs as normal
   Company notes** (e.g. a post from a lab or vendor → a company note for that org
   + the author as a person), matched by basename or created, appearing in the normal People/Companies
   bases — no separate tag or view.
3. **`wiki/` (synthesis, LLM-owned).** Topic folders `wiki/<topic>/` each with an
   `_index.md`, article notes ending in `## Key Takeaways`, a front-door
   `wiki/_master-index.md`, and an append-only `wiki/log.md`. Wiki pages carry **no
   tags and no frontmatter** (bar a topic `_index.md`'s single `Status:`), which is
   what keeps them out of every base, kanban, and the daily rundown.

Ingest is **manual** ("process my resources" / "compile") **and scheduled** (the
`obsidian-process-resources` auto-scan). Google-Docs/PDF sources often clip empty
or as UI chrome — flag those (⚠️ in the topic's provenance table), never fabricate.
`raw/`, `Resources/`, and `wiki/` have **no** Templater folder-template mappings
(skill-owned, like Sessions/Meetings).

## Golden rules

1. **Frontmatter links are quoted wikilinks**: `Company: "[[Acme]]"`; multi-value
   = a YAML list of `- "[[X]]"`. Plain text does NOT create a backlink.
2. **Wikilink target = the other note's basename** (folder-independent). One note
   per entity; reconcile duplicates to the fullest name and fix backlinks with an
   exact token replace.
3. **People / Projects live in `<Company>` subfolders**; Companies is flat.
   **Tasks are flat in `Tasks/Task Notes/`** (no per-project subfolders — the base
   organizes by the `Project` property). `Meetings/` has three subfolders —
   `Inbox/` (Granola sync dump, transient, flat), `Processed Meetings/`, and
   `Transcripts/`; the latter two bucket their notes into `<YYYY>-wk.<WW>/`
   ISO-week subfolders (e.g. `2026-wk.29/`), a note and its transcript sharing the
   same bucket. `Tasks/` also holds `Rundowns/` — digest notes tagged
   `Rundown`, never `Task` (they must stay out of the Tasks base and kanban boards),
   and `Pending/`, holding the single **untagged** `Proposed Tasks.md` staging queue of
   meeting-derived task proposals awaiting the user's validation (a proposal is not a
   task: never tagged `Task`, never numbered, never on a board).
4. **Divisions** are plain-text org units: a company lists its divisions; a person
   picks one (identical string). Blank for orgs with no division breakdown.
   **Your own company's projects always carry `Division: internal` or
   `Division: personal`** (never a bare `Company: "[[Acme]]"` with no Division) to
   separate the company's own internal initiatives from your personal projects, so
   the Projects base reads them as "Acme [internal]" / "Acme [personal]". This
   project-level Division applies to your own company only and is distinct from the
   People division taxonomy (org units like Engineering/Sales); client projects
   don't use it. Set the company name in `Templates/Project.md` (`OWN_COMPANY`).
5. **Tasks**: `Task Number` always quoted (`"1"`); `Status` ∈ {Not Started,
   In Progress, Completed, Discarded}; no sub-task hierarchy — every task is a
   flat note; `Project: General task` (plain text) when none. `Priority` is an
   optional `High`/`Medium`/`Low` (blank = unset) used to decide what to book
   first — never mandatory. `Effort` is the
   estimated minutes to do the task, always a multiple of 15 (`15`, `30`, `45`, …),
   blank when unknown — it's what calendar slots are sized from. `Booked ?` is a
   plain `Y`/`N` flag (default `N`) tracking whether a calendar slot has been
   booked for the task; flip to `Y` once booked. **`Status`
   frontmatter is the single source of truth** — the kanban boards
   (`Tasks/Tasks Kanban.base` and each project dashboard's Board) are TaskNotes
   views over it: they pick up tasks and drag-drop writes `Status` back. Never
   maintain board files by hand. **Projects** likewise carry a sequential
   `Project Number` (always quoted, like `Task Number`; one global sequence across
   all companies), auto-assigned by the Project template as `(highest existing)+1`.
6. **Meetings** base is folder-filtered on `Meetings/Processed Meetings/` (not
   tag-based). `Meetings/Inbox/` (the Granola inbox) is intentionally excluded
   from Templater folder-templates because Granola auto-dumps notes/transcripts
   there — a folder template would corrupt them, and the sync overwrites that
   folder anyway. Deleting a raw inbox file doesn't stick while its meeting is in
   Granola's sync window (the sync re-dumps it ~every 30 min); that's expected —
   the base only indexes `Processed Meetings/`, and the auto-processor re-files.
7. **Never edit `.base` view files** unless asked; **prefer a blank property over
   an invented value**; after edits, **verify no broken wikilinks**.
8. **`raw/` is immutable** — read, never edit/rename/move/delete a clipping there.
   **`wiki/` pages and `raw/` clippings never carry the CRM tags** (`Task`,
   `Meeting`, `Session`, and wiki pages also not `People`/`Company`/`Project`), so
   they stay out of every base, kanban, and the rundown. Resource notes are the one
   exception: tagged `Resource`, they belong to the Resources base.

## Creating notes

Templater (`Templates/`) scaffolds new notes and auto-files them:
- A new note in `People/`, `Companies/`, `Projects/`, or `Tasks/` runs the matching
  template (folder templates, trigger-on-create).
- The `Meeting` template is manual (Insert Template) — never folder-mapped — and
  files into `Meetings/Processed Meetings/`.
- `Sessions/` has **no folder template** (like `Meetings/`): session notes are
  created fully-formed by `obsidian-log-session`; a trigger-on-create template
  would clobber them. Reports produced at session close live with the session
  (a section in the session note, or a companion note in `Sessions/<Project>/`)
  — there is **no top-level Reports folder**.
- `Tasks/Rundowns/` and `Tasks/Pending/` are **shielded from the Task folder template**
  (a no-op `Templates/Noop.md` mapping each plus guards inside `Templates/Task.md`):
  rundown notes are generated fully-formed by `obsidian-daily-rundown`, and the
  proposal queue by `obsidian-process-meeting` / the meeting auto-scan.
- `raw/`, `Resources/`, and `wiki/` have **no folder template** (like
  `Sessions/`/`Meetings/`): `raw/` is filled by the Web Clipper, and Resource/wiki
  notes are created fully-formed by `obsidian-process-resource`.

**Fresh-vault order:** create **Companies** (with their Divisions) first, then
People/Projects, then Tasks.

## Rebuilding elsewhere

This repo *is* the empty scaffolding: clone it, open the folder as an Obsidian
vault, and start entering data. See `README.md` for the setup steps.

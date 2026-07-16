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
  `Meetings/` → crosslinked note + extracted tasks (task-entry METHOD 2).
- `.claude/skills/obsidian-log-session/SKILL.md` — user finishes a work session →
  session note in `Sessions/<Project>/` (what was done, decisions, task statuses).
- `.claude/skills/obsidian-daily-rundown/SKILL.md` — daily digest of open tasks
  by due date → `Tasks/Rundowns/YYYY-MM-DD Rundown.md` (a scheduled task runs it
  every day at 9am; also on demand: "what's on my plate today").

## Bases, folders, membership

| Base | Folder | In the base when… | Key properties |
|---|---|---|---|
| Companies | `Companies/` | tagged `Company` | Division, Stakeholders, Projects, Notes |
| People | `People/<Company>/` | tagged `People` | Company, Division, Role, Email, LinkedIn, Notes |
| Projects | `Projects/<Company>/` | tagged `Project` | Company, Start/End Date, Status, Link |
| Tasks | `Tasks/Task Notes/` (flat) | tagged `Task` | Task Number, Status, Project, Stakeholders, Company, Created/Due Date, Notes, Links |
| Meetings | `Meetings/Processed Meetings/` | is in the `Meetings/Processed Meetings/` folder | Date, Attendees, Company, Project, Links, Transcript |

Meeting notes are filed with a **`YYYYMMDD ` date prefix** (`20260521 GTM session.md`)
so the Meetings base reads chronologically; the base surfaces a **Transcript**
column linking each note to its raw transcript. Transcripts themselves are not
date-prefixed.
| Sessions | `Sessions/<Project>/` | tagged `Session` | Date, Project, Company, Tasks Completed, Tasks In Progress |

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
the vault) and listing them in a `## People mentioned` section, without editing
Granola's verbatim body. On every filed copy it renames the
`granola_id` frontmatter key to **`granola_ref`**: the plugin owns any file carrying
`granola_id` **vault-wide** (it re-indexes and clobbers it wherever it sits), so the
rename — not just the move — is what makes filed notes permanently safe. Never
enrich a note in place while it still has `granola_id`. Dedup matches a raw note's
`granola_id` against filed `granola_ref`; the meeting date is Granola's `created`
(when it happened), never `updated`.

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
change its field mapping.

## Golden rules

1. **Frontmatter links are quoted wikilinks**: `Company: "[[Acme]]"`; multi-value
   = a YAML list of `- "[[X]]"`. Plain text does NOT create a backlink.
2. **Wikilink target = the other note's basename** (folder-independent). One note
   per entity; reconcile duplicates to the fullest name and fix backlinks with an
   exact token replace.
3. **People / Projects live in `<Company>` subfolders**; Companies is flat.
   **Tasks are flat in `Tasks/Task Notes/`** (no per-project subfolders — the base
   organizes by the `Project` property). `Meetings/` has three flat subfolders —
   `Inbox/` (Granola sync dump, transient), `Processed Meetings/`, and
   `Transcripts/`. `Tasks/` also holds `Rundowns/` — digest notes tagged
   `Rundown`, never `Task` (they must stay out of the Tasks base and kanban boards).
4. **Divisions** are plain-text org units: a company lists its divisions; a person
   picks one (identical string). Blank for orgs with no division breakdown.
5. **Tasks**: `Task Number` always quoted (`"1"`); `Status` ∈ {Not Started,
   In Progress, Completed, Discarded}; no sub-task hierarchy — every task is a
   flat note; `Project: General task` (plain text) when none. **`Status`
   frontmatter is the single source of truth** — the kanban boards
   (`Tasks/Tasks Kanban.base` and each project dashboard's Board) are TaskNotes
   views over it: they pick up tasks and drag-drop writes `Status` back. Never
   maintain board files by hand.
6. **Meetings** base is folder-filtered on `Meetings/Processed Meetings/` (not
   tag-based). `Meetings/Inbox/` (the Granola inbox) is intentionally excluded
   from Templater folder-templates because Granola auto-dumps notes/transcripts
   there — a folder template would corrupt them, and the sync overwrites that
   folder anyway. Deleting a raw inbox file doesn't stick while its meeting is in
   Granola's sync window (the sync re-dumps it ~every 30 min); that's expected —
   the base only indexes `Processed Meetings/`, and the auto-processor re-files.
7. **Never edit `.base` view files** unless asked; **prefer a blank property over
   an invented value**; after edits, **verify no broken wikilinks**.

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
- `Tasks/Rundowns/` is **shielded from the Task folder template** (a no-op
  `Templates/Noop.md` mapping plus a guard inside `Templates/Task.md`): rundown
  notes are generated fully-formed by `obsidian-daily-rundown`.

**Fresh-vault order:** create **Companies** (with their Divisions) first, then
People/Projects, then Tasks.

## Rebuilding elsewhere

`SETUP/setup-vault.sh` scaffolds this whole structure (empty, no data) on another
machine. See `SETUP/README.md`.

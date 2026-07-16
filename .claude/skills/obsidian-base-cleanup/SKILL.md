---
name: obsidian-base-cleanup
description: >-
  Clean up and crosslink this Obsidian vault's Bases (People, Companies,
  Projects, Tasks). Use whenever the user asks to "clean up the vault", "clean
  up the bases", crosslink or backlink People/Companies/Projects, add missing
  people/companies/projects mentioned in notes, fix `[[wikilink]]` properties,
  or reorganize notes into the right folders. Also trigger when a note (often a
  company handbook) mentions new people, projects, or companies that aren't yet
  represented as their own notes and linked. Handles converting plain-text
  frontmatter properties into resolvable wikilinks and moving project notes into
  their company subfolder.
---

# Obsidian Base Cleanup

This vault is organized around Obsidian **Bases** — table views over notes that
share a tag. Cleanup means three things: every entity (person, company, project)
has its own note; every relationship between them is a resolvable `[[wikilink]]`
in frontmatter; and every note lives in the right folder. Do these in the order
below.

## The vault model

Five Bases. Four are filtered by a tag (a note only appears if it carries the
tag — **the tag is load-bearing, never drop it**); Meetings is filtered by folder.

| Base | Folder | Filter | Key properties |
|---|---|---|---|
| People | `People/<Company>/` | tag `People` | `Company`, `Division`, `Role`, `Email`, `LinkedIn`, `Notes` |
| Companies | `Companies/` | tag `Company` | `Division`, `Stakeholders`, `Projects`, `Notes` |
| Projects | `Projects/<Company>/` | tag `Project` | `Company`, `Start Date`, `End Date`, `Status`, `Link` |
| Tasks | `Tasks/Task Notes/` (flat) | tag `Task` | `Task Number`, `Status`, `Project`, `Stakeholders`, `Company`, `Created/Due Date`, `Notes`, `Links` (see `obsidian-create-task`) |
| Meetings | `Meetings/Processed Meetings/` | folder `file.inFolder("Meetings/Processed Meetings")` | `Date`, `Attendees`, `Company`, `Project`, `Links`, `Transcript` |
| Sessions | `Sessions/<Project>/` | tag `Session` | `Date`, `Project`, `Company`, `Tasks Completed`, `Tasks In Progress` (see `obsidian-log-session`) |

New notes are created through **Templater** (see below), which stamps the right
frontmatter and files the note automatically. When creating notes yourself
(scripted bulk work), match the shape the templates produce. The `.base` files
are the view definitions; **don't edit them** unless the user asks to change
columns/filters/sort. Note: Bases reference frontmatter properties by their bare
name in `sort`/`order` (e.g. `property: Company`, not `note.Company`).

### Templater

The vault uses the Templater plugin, configured in
`.obsidian/plugins/templater-obsidian/data.json`:

- **Template folder**: `Templates/` — holds `Person.md`, `Company.md`,
  `Project.md`, `Task.md`, `Meeting.md`.
- **Folder templates** (with `trigger_on_file_creation: true`): a new note in
  `People/`, `Companies/`, `Projects/`, or `Tasks/` (or any subfolder) auto-runs
  the matching template.

Each template is written as a single leading `<%* … %>` execution block so the
**template file itself has no frontmatter** — that's deliberate: a literal
`tags: [People]` in a template would make the template show up inside the People
Base. The block prompts for values, writes the frontmatter via `tR`, then calls
`tp.file.move(...)` to file the note in the correct `<Company>` subfolder.

**Meetings & Granola.** Everything meeting-related lives under `Meetings/`. The
Granola sync auto-dumps raw notes **and** raw transcripts into `Meetings/Inbox/` —
a **quarantined**, transient inbox the sync re-dumps on every run (kept in its own
subfolder so the churn stays out of the `Meetings/` root). The
`obsidian-process-meeting` skill **moves** each raw note into
`Meetings/Processed Meetings/` (the clean, crosslinked note that backs the Meetings
Base) and its transcript into `Meetings/Transcripts/` (tagged `RawTranscript`), then
deletes the inbox copies. On every filed copy it renames the `granola_id`
frontmatter key to **`granola_ref`** — the plugin owns any file carrying
`granola_id` **vault-wide** and will re-index/clobber it back into `Meetings/Inbox/`,
so the rename (not just the move) is what keeps filed notes safe; dedup matches a
raw note's `granola_id` to filed `granola_ref`. The Base therefore filters by
**folder** (`file.inFolder("Meetings/Processed Meetings")`), not by a tag.
`Meetings/Inbox/` is deliberately **left out of the Templater folder-templates**:
with `trigger_on_file_creation` on, a folder template would fire on every file
Granola drops in and corrupt it. The `Meeting` template is manual-only (apply via
"Insert Template") and files into `Meetings/Processed Meetings/`. Never enrich a note
in place while it still carries `granola_id` — the next sync clobbers it; always
process into `Meetings/Processed Meetings/`. (Note: a raw file re-appearing in
`Meetings/Inbox/` after you delete it is expected — Granola re-dumps any meeting
still in its sync window; the auto-processor re-files it and the Base never shows a
duplicate.)

**Project dashboards, TaskNotes & Sessions.** Every project note is its own
dashboard: `## Overview`, `## Tasks` = `![[All Tasks.base#Project Board]]` (a
`tasknotesKanban` view — the **TaskNotes** plugin, remapped to this vault's
schema: tag `Task`, `Status`, `Due Date`, `Project`, `Created Date` — don't change
its field mapping), `## Meetings` = `![[Meetings.base#Project Meetings]]`,
`## Sessions` = `![[Sessions.base#Project Sessions]]`, and `## Key Decisions`.
The `Project …` views are defined once in the **master bases** and filter on
`file.hasLink(this.file)` — `this` resolves to the note embedding the view, so
the same view serves every dashboard (and shows empty when the master base is
opened directly; that's expected, not a bug). They are **load-bearing: never
rename or delete them**, and edit them only to change every dashboard at once
(All Tasks.base also has a `Project Table` variant). Task `Status` frontmatter is
the **single source of truth**;
the boards (`Tasks/Tasks Kanban.base` and each dashboard's Board) are views over
it — dragging a card writes `Status` (plus TaskNotes bookkeeping like
`completedDate`/`dateModified`, which is fine to leave) back into the note. There
is no board file to maintain. Session notes are created fully-formed by
`obsidian-log-session` into `Sessions/<Project>/`; like `Meetings/`, the
`Sessions/` folder is deliberately **left out of the Templater folder-templates**
so trigger-on-create can't clobber them.

The clever part to preserve: the Person template reads the chosen **company's own
`Division` list** from its frontmatter (`app.metadataCache.getFileCache(...)
.frontmatter.Division`) and offers exactly those as the division options — so
divisions stay consistent without hardcoding. The Project template files into
`Projects/<Company>/`; the Task template inherits its `Company` from the chosen
project. When you change the folder/property conventions, update these templates
to match.

### Folder convention

Project notes are grouped into a subfolder named after their company, e.g.
`Projects/Acme/Widget.md`. When you create or find a project note (including loose
ones dropped in the vault root), move it to `Projects/<its Company>/`. Projects
with no company go directly in `Projects/`. People notes are likewise grouped into
`People/<Company>/` subfolders, matching each person's `Company` property. The
`.base` view file and `Templates/` stay put. Company notes are flat in `Companies/`.
**Task notes are the exception: they are all flat in `Tasks/Task Notes/`** — no
per-project subfolders (the Tasks base organizes them by the `Project` property,
so the folder plays no role).

**`External` people.** For someone who belongs to no tracked org — prospective
hires/candidates, or contacts who aren't your own company and aren't a client — set
`Company: External` (plain text, **never a wikilink**, exactly like `General task`
for tasks) and file them under `People/External/`. `External` is a catch-all
label, **not** a real company: never create an `External` company note and it
never appears in the Companies base. If such a person is later hired or tied to a
real org, switch their `Company` to that org's wikilink and move them into the
matching `People/<Company>/` folder.

### Divisions

Large clients are internally divided into **divisions** (org units). This is
tracked with a `Division` property, as **plain text** (not wikilinks — divisions
don't have their own notes):

- On a **company** note, `Division` is the list of divisions that exist inside
  that company.
- On a **person** note, `Division` is the single division they sit in (must be
  one of the strings from their company's `Division` list — keep the strings
  identical so the Base can group/filter on them).

People whose `Company` is the user's own org (or any company without a division
breakdown) leave `Division` blank. When cleaning up, read the company handbook's
"how it's structured" / "people" sections — that's the source of truth for both
the division list and each person's division.

## How linking works (the crucial detail)

Obsidian only renders a property value as a real link — and only then does it
create a backlink — if the value is a wikilink. In YAML frontmatter a wikilink
**must be quoted** or the YAML is invalid:

```yaml
Company: "[[Acme]]"
```

For multi-value properties (Stakeholders, Projects), use a YAML list of quoted
wikilinks:

```yaml
Stakeholders:
  - "[[Jane Doe]]"
  - "[[John Roe]]"
Projects:
  - "[[Widget]]"
```

Plain text like `Stakeholders: Jane, John` does **not** link — converting these
to wikilinks is the core of the cleanup. The wikilink target is the other note's
**file basename** (no extension, no folder), so `[[Widget]]` resolves to
`Projects/Acme/Widget.md` from anywhere.

## Workflow

Work in this order so that by the time you write a link, its target exists.

### 1. Read everything and inventory
Read all `.md` notes. Build a mental list of existing People, Companies, and
Projects (by file basename). Company handbooks (long note bodies) are the richest
source of un-noted entities — read them closely.

### 2. Extract mentioned entities
Scan note bodies and properties for **named people, projects, and companies**
that don't yet have their own note. Capture, for each, whatever the source tells
you: a person's company, role, and a one-line context; a project's company,
status, and dates; a company's nature.

Judgment calls:
- **People**: add every *named individual* who's a real contact/stakeholder/
  end-user. Include peripheral ones but keep their role/notes honest ("peripheral",
  "former sponsor", etc.).
- **Same person, multiple sources**: the same individual (especially your own
  team) often appears in several company handbooks — sometimes first-name-only in
  one and full-name in another. Never create a second note. Reconcile to one note
  per person, preferring the fullest name as the filename, enrich it with facts
  from every source, and update existing backlinks to the new name. To rename a
  note safely, grep for `[[OldName]]` across the vault, then do an exact
  token-replace (`perl -i -pe 's/\Q[[OldName]]\E/[[New Name]]/g'`) — because links
  resolve by basename, this keeps every backlink intact. A person's `Company` is
  their *employer* (so shared internal/vendor staff live under their own company
  folder, not the client's), even when the note lives in a client's handbook.
- **Consultancies / third parties** (a consultancy contracted to a client): a
  `Company`-tagged note like any other org, with those people filed under
  `People/<Consultancy>/`.
- **Companies**: every org — clients, the user's own company, vendors,
  consultancies — is tagged `Company` and appears in the Companies Base. Don't
  add tools or platforms that aren't real orgs. (There is no separate `Client`
  tag; the base shows all companies.)
- **Projects**: workflows/deliverables tied to a company are projects. Infer the
  company from context.

### 3. Create the missing notes
Create each new note with the correct tag and properties. Fill properties from the
source; leave genuinely unknown ones blank rather than inventing. Use the person's
display name as the filename. Add a short body paragraph for substantive entities
linking related notes — this seeds the backlink graph.

For bulk creation (many people at once), a single shell script with a small
heredoc helper is far more efficient than dozens of individual writes.

### 4. Crosslink existing notes
Convert every plain-text relationship property into quoted wikilinks:
- People `Company` → `"[[Company]]"`.
- Company `Stakeholders` → list of the client-side people; `Projects` → list of
  that company's projects. (Keep the user's internal team out of a *client's*
  Stakeholders — note them in `Notes` instead.)
- Project `Company` → `"[[Company]]"`.

When editing a note that has a large body (a handbook), edit **only the
frontmatter** and leave the body untouched.

### 5. Reorganize folders
Move project notes into `Projects/<Company>/` and people into `People/<Company>/`,
creating subfolders as needed. Delete the old-location copy after writing the new.

### 6. Verify no broken links
Every wikilink target must be an existing note basename. Confirm with:

```bash
cd <vault-root>
find . -not -path '*/.*' -name '*.md' | sed 's#.*/##; s#\.md##' | sort -u > /tmp/notes.txt
grep -rho '\[\[[^]]*\]\]' --include='*.md' . | sed 's/\[\[//; s/\]\]//; s/|.*//' | sort -u > /tmp/links.txt
comm -23 /tmp/links.txt /tmp/notes.txt   # prints any link with no matching note — should be empty
```

Resolve anything that prints: either create the missing note or fix the typo.
Then summarize for the user what was added, linked, and moved.

## Notes on safety
- Preserve tags and existing body content. The tag drives Base membership; body
  content (especially handbooks) is hand-written and valuable.
- Don't touch `.base` view files unless asked.
- Prefer leaving a property blank over guessing a value.

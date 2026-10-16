---
name: obsidian-base-cleanup
description: >-
  Clean up and crosslink this Obsidian vault's Bases (People, Companies,
  Projects, Initiatives, Tasks). Use whenever the user asks to "clean up the vault", "clean
  up the bases", crosslink or backlink People/Companies/Projects, add missing
  people/companies/projects mentioned in notes, fix `[[wikilink]]` properties,
  or reorganize notes into the right folders. Also trigger when a note (often a
  company handbook) mentions new people, projects, or companies that aren't yet
  represented as their own notes and linked. Handles converting plain-text
  frontmatter properties into resolvable wikilinks and filing project notes into
  the right client or internal projects folder.
---

# Obsidian Base Cleanup

This vault is organized around Obsidian **Bases** — table views over notes that
share a tag. Cleanup means three things: every entity (person, company, project,
initiative) has its own note; every relationship between them is a resolvable
`[[wikilink]]` in frontmatter; and every note lives in the right folder. Do these
in the order below.

## The vault model

Every Base filters by a tag (a note only appears if it carries the tag — **the
tag is load-bearing, never drop it**), except Meetings, which filters by folder.

| Base | Folder | Filter | Key properties |
|---|---|---|---|
| People | `People/<Company>/` | tag `People` | `Company`, `Division`, `Role`, `Email`, `LinkedIn`, `Notes` |
| Companies | `Companies/` | tag `Company` | `Division`, `Stakeholders`, `Projects`, `Notes`, `Slack Channels`, `Slack DMs` |
| Projects | `Projects/Client projects/<Company>/`, or `Projects/Internal projects/` (flat) | tag `Project` | `Project Number` (quoted, one global sequence, template auto-assigns `max+1`), `Company`, `Division` (your own company only: internal/personal), `Initiative`, `Start/End Date`, `Status`, `Stakeholders`, `Related Tasks`, `Link`, `Slack Channels`, `Slack DMs` |
| Initiatives | `Initiatives/` (flat) | tag `Initiative` | `Initiative Number` (quoted, its own sequence), `Company`, `Stakeholders`, `Related Projects`, `Related Tasks`, `Link`, `Slack Channels`, `Slack DMs`. No `Status`, no dates. |
| Tasks | `Tasks/Task Notes/` (flat) | tag `Task` | `Task Number`, `Status`, `Project`, `Initiative`, `Stakeholders`, `Company`, `Created/Due Date`, `Notes`, `Links` (see `obsidian-create-task`) |
| Meetings | `Meetings/Processed Meetings/` | folder `file.inFolder("Meetings/Processed Meetings")` | `Date`, `Attendees`, `Company`, `Project`, `Initiative`, `Links`, `Transcript` |
| Sessions | `Sessions/<Project or Initiative>/` | tag `Session` | `Date`, `Project`, `Initiative`, `Company`, `Tasks Completed`, `Tasks In Progress` (see `obsidian-log-session`) |

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
  `Project.md`, `Initiative.md`, `Task.md`, `Meeting.md`.
- **Folder templates** (with `trigger_on_file_creation: true`): a new note in
  `People/`, `Companies/`, `Projects/`, `Initiatives/`, or `Tasks/` (or any
  subfolder) auto-runs the matching template.

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

**Creating a project: always ask for the target end date and milestones.** A new
project note is not finished until it has a target `End Date` and a
`## Milestones` list. Ask the user for both, one project at a time. You may suggest
candidate milestones drawn from the Overview, linked meetings, and open tasks,
but **never write one without his explicit authority** — an unanswered
suggestion creates nothing, and an empty `## Milestones` beats an invented one.
Milestones are checkboxes, `- [ ] <name> — YYYY-MM-DD`, one per line. An
initiative has no `End Date` to ask for, being open-ended, but its
`## Milestones` list follows the same rule: suggest, never write unasked.

**Project dashboards, TaskNotes & Sessions.** Every project note is its own
dashboard: `## Overview`, `## Tasks` = `![[All Tasks.base#Project Table]]`,
`## Milestones` (prose checkboxes, directly after `## Tasks`),
`## Meetings` = `![[Meetings.base#Project Meetings]]`,
`## Sessions` = `![[Sessions.base#Project Sessions]]`,
`## Resources` = `![[Resources.base#Project Resources]]`, and `## Key Decisions`.
The `Project …` views are defined once in the **master bases** and filter on
`file.hasLink(this.file)` — `this` resolves to the note embedding the view, so
the same view serves every dashboard (and shows empty when the master base is
opened directly; that's expected, not a bug). They are **load-bearing: never
rename or delete them**, and edit them only to change every dashboard at once.
`All Tasks.base` also keeps a `Project Board` `tasknotesKanban` view; it is no
longer embedded in any dashboard (the table reads better in a note), so leave it
in place but don't add it back to a note body.

The **TaskNotes** plugin is remapped to this vault's schema (tag `Task`,
`Status`, `Due Date`, `Project`, `Created Date` — don't change its field
mapping). Task `Status` frontmatter is the **single source of truth**; the boards
(`Tasks/Tasks Kanban.base` and the `Project Board` view) are views over it —
dragging a card writes `Status` (plus TaskNotes bookkeeping like
`completedDate`/`dateModified`, which is fine to leave) back into the note. There
is no board file to maintain. **TaskNotes maps exactly one property as its
project field (`Project`)**, so a task that belongs to an initiative instead
shows a blank project in TaskNotes' own UI and in the kanban project field. That
is cosmetic: every Bases view still picks the task up, because those views filter
on `file.hasLink(this.file)`, which matches a link in **any** frontmatter
property.

Session notes are created fully-formed by `obsidian-log-session` into
`Sessions/<Project or Initiative>/`; like `Meetings/`, the `Sessions/` folder is
deliberately **left out of the Templater folder-templates** so trigger-on-create
can't clobber them.

The clever part to preserve: the Person template reads the chosen **company's own
`Division` list** from its frontmatter (`app.metadataCache.getFileCache(...)
.frontmatter.Division`) and offers exactly those as the division options — so
divisions stay consistent without hardcoding. The Project template files into
`Projects/Client projects/<Company>/`, or `Projects/Internal projects/` when the
company is your own (`OWN_COMPANY`), and asks for an optional parent initiative. The Task template
offers projects **and** initiatives in one picker and inherits its `Company` from
whichever was chosen. When you change the folder/property conventions, update
these templates to match.

### Initiatives

An **initiative** is an open-ended container, wider than a project: a standing
track of work (a GTM motion, a hiring track, a conference, a recurring review)
that has no fixed scope and therefore **no `Start Date` and no `End Date`**. A
project is the bounded thing inside it. Initiative notes are tagged `Initiative`,
flat in `Initiatives/`, and carry their own `Initiative Number` sequence,
separate from `Project Number`.

Frontmatter, in this key order:

```yaml
---
tags:
  - Initiative
Initiative Number: "6"       # ALWAYS quoted; its own sequence, (highest) + 1
Company: "[[Acme]]"
Status: In Progress
Stakeholders:
  - "[[Jane Doe]]"
Related Projects:            # quoted wikilinks to project notes
  - "[[Widget Benchmark]]"
Related Tasks:               # ALIASED wikilinks that display the task number
  - "[[Review the one-pager|#194]]"
Link:
Slack Channels:              # quoted, with the #: an unquoted # is a YAML comment
  - "#acme-widget-launch"
Slack DMs:                   # quoted person wikilinks
  - "[[Sarah Chen]]"
---
```

`Related Tasks` uses the aliased form `- "[[<task title>|#<Task Number>]]"` so
the base column reads as a list of task numbers while the link still resolves to
the note's basename. Project notes carry the same `Related Tasks` list.

The body is the initiative's dashboard, in this order: `## Overview`,
`## Tasks` (`![[All Tasks.base#Initiative Table]]`, grouped by Status),
`## Milestones`, `## Related Projects` (`### Internal` embedding
`![[Projects.base#Initiative Internal Projects]]`, `### Client` embedding
`![[Projects.base#Initiative Client Projects]]`), `## Next Steps` (hand-kept
checkbox list, `- [ ] <what> (waiting on [[Person]], since YYYY-MM-DD)`), `## Meetings`
(`![[Meetings.base#Project Meetings]]`), `## Sessions`
(`![[Sessions.base#Project Sessions]]`), `## Resources`
(`![[Resources.base#Project Resources]]`), `## Key Decisions`. The embedded views
are the same central `this.file` views the project dashboards use, so an
initiative needs no registration: anything whose frontmatter wikilinks it shows
up. `Initiatives.base` also carries a `Company Initiatives` view
(tag + `hasLink(this.file)`) for embedding on a company note.

**Never invent an initiative.** Unlike people and companies, initiatives are not
extracted from prose: they are containers the user decides on. Link an existing one,
or leave the field blank and say so.

### Folder convention

Project notes are split by whose project it is:

- **Client projects** go into a subfolder named after their company:
  `Projects/Client projects/Acme/Widget.md`.
- **Internal (your own company's) projects** are **flat** in `Projects/Internal projects/`
  — no company subfolder, since they all belong to your company.

`Projects/Projects.base` stays at the top of `Projects/`. When you create or find
a project note (including loose ones dropped in the vault root), move it to
whichever of the two homes its `Company` implies. **Initiative notes are flat in
`Initiatives/`.** People notes are grouped into `People/<Company>/` subfolders,
matching each person's `Company` property. The `.base` view files and `Templates/`
stay put. Company notes are flat in `Companies/`.
**Task notes are flat in `Tasks/Task Notes/`** — no per-project subfolders (the
Tasks base organizes them by the `Project` and `Initiative` properties, so the
folder plays no role).

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

**Your own company's projects are the one place `Division` is used on a project
note.** Every project whose `Company` is your own company (say `Acme`) must also
carry `Division: internal` or `Division: personal` (the company's own internal
initiatives vs. your personal projects), so the Projects base reads them as
"Acme [internal]" / "Acme [personal]". Never leave one of those projects with no
`Division`. This classifier reuses the `Division` property name but is separate
from the People division taxonomy (Engineering/Sales/etc.), and client projects
don't set it. `Templates/Project.md` holds the company name in `OWN_COMPANY`; set
it to `""` to switch the whole rule off.

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
`Projects/Client projects/Acme/Widget.md` from anywhere. An **aliased** wikilink,
`"[[Widget|#12]]"`, links the same note but displays `#12`; that is the form
`Related Tasks` uses.

## Workflow

Work in this order so that by the time you write a link, its target exists.

### 1. Read everything and inventory
Read all `.md` notes. Build a mental list of existing People, Companies,
Projects, and Initiatives (by file basename). Company handbooks (long note
bodies) are the richest source of un-noted entities — read them closely.

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
  company from context; it decides the home folder (a client company's own
  subfolder, or flat under `Internal projects/` for your own company's work).
- **Initiatives**: never created from prose. Link an existing one or leave the
  field blank (see Initiatives above).

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
- Project `Company` → `"[[Company]]"`; `Initiative` → `"[[Initiative]]"` when the
  project sits under one, else blank.
- Initiative `Related Projects` → list of `"[[Project]]"` links.
- Project and initiative `Related Tasks` → list of **aliased** links,
  `- "[[<task title>|#<Task Number>]]"`. Both sides of a container↔task
  relationship are written: the task's `Project`/`Initiative` key, and the
  container's `Related Tasks` entry. Keep them in agreement; a task listed in
  `Related Tasks` whose own frontmatter points elsewhere is a bug.

When editing a note that has a large body (a handbook), edit **only the
frontmatter** and leave the body untouched.

### 5. Reorganize folders
Move client project notes into `Projects/Client projects/<Company>/`, your own company's
project notes flat into `Projects/Internal projects/`, initiative notes flat into
`Initiatives/`, and people into `People/<Company>/`, creating subfolders as
needed. Delete the old-location copy after writing the new.

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

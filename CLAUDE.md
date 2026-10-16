# Personal Assistant Vault: guide for Claude

A lightweight CRM plus a project and task system, built on three Obsidian
plugins:

- **Bases**: table views over notes.
- **Templater**: note scaffolding and auto-filing.
- **TaskNotes**: drag-drop kanban views over task notes.

This file holds the schema and the always-on rules. The **skills hold the
procedures.** Read the relevant skill before doing the work. Do not rely on this
file for step-by-step detail.

## Skills: which one to read

| Skill (in `.claude/skills/<name>/SKILL.md`) | Read it when |
|---|---|
| `obsidian-base-cleanup` | Fixing entities, crosslinks, folders, divisions, Templater. |
| `obsidian-create-task` | Turning natural language into a task. Task-entry METHOD 1. |
| `obsidian-process-meeting` | Processing a Granola note in `Meetings/`. Task-entry METHOD 2. |
| `obsidian-log-session` | The user finishes a work session. |
| `obsidian-daily-rundown` | "What's on my plate." Also scheduled daily at 9am. |
| `obsidian-book-tasks` | "Book my tasks." On demand only, never at task creation. |
| `obsidian-process-resource` | "Process my resources" / "compile". Also auto-scans `raw/`. |

## Bases, folders, membership

| Base | Folder | In the base when… | Key properties |
|---|---|---|---|
| Companies | `Companies/` | tagged `Company` | Division, Stakeholders, Projects, Meetings, Notes, Slack Channels, Slack DMs |
| People | `People/<Company>/` | tagged `People` | Company, Division, Role, Email, LinkedIn, Notes |
| Projects | `Projects/Client projects/<Company>/` and `Projects/Internal projects/` | tagged `Project` | Project Number, Company, Initiative, Start/End Date, Status, Related Tasks, Link, Slack Channels, Slack DMs |
| Initiatives | `Initiatives/` (flat) | tagged `Initiative` | Initiative Number, Company, Stakeholders, Related Projects, Related Tasks, Link, Slack Channels, Slack DMs |
| Tasks | `Tasks/Task Notes/` (flat) | tagged `Task` | Task Number, Status, Project, Initiative, Stakeholders, Company, Created/Due Date, Priority, Effort, Booked ?, Notes, Links |
| Meetings | `Meetings/Processed Meetings/` | it sits in that folder (not by tag) | Date, Attendees, Company, Project, Initiative, Links, Transcript |
| Sessions | `Sessions/<Project or Initiative>/` | tagged `Session` | Date, Project, Initiative, Company, Tasks Completed, Tasks In Progress |
| Resources | `Resources/` (flat) | tagged `Resource` | Title, Link, Type, Author, Company, Project, Initiative, Date, Note, Raw |

`raw/` and `wiki/` are **not** bases. `raw/` is the immutable Web Clipper inbox:
clippings are named `YYYY-MM-DD-<Title>.md` and carry the clipper's own
*Clippings* frontmatter. Web Clipper setup is browser-side, documented in
`raw/README.md`. `wiki/` pages are deliberately untagged, which keeps them out
of every base, kanban, and rundown.

**Naming.** Processed meeting notes take a `YYYYMMDD ` date prefix, as in
`20260521 GTM session.md`, so the base reads chronologically. Transcripts take
no prefix. Session notes are named `YYYY-MM-DD <Project> Session`.

**Companies and people.** Every org gets a `Company` note: clients, your own
company, vendors, consultancies. People who belong to no tracked org get
`Company: External` as plain text and live in `People/External/`. `External` is
a catch-all label, never a real company note.

**Slack sources.** Companies, projects and initiatives say where in Slack to
find the latest on them, with two keys at the end of the frontmatter:

- `Slack Channels`: quoted channel names with the `#`, as in
  `- "#acme-widget-launch"`. The quotes are required: an unquoted `#` starts a YAML
  comment and the value is lost.
- `Slack DMs`: quoted person wikilinks, as in `- "[[Sarah Chen]]"`, for
  people whose DMs carry news on it.

A channel can sit on more than one note: a project's channel goes on the
project and on the initiative it serves. To get the latest on an initiative,
read its own channels plus those of its `Related Projects`, deduplicated. A
scoping channel goes on the company until the SOW has a project note, then on
the project as well. Leave both keys blank when nothing is known; never guess a
channel name.

**Project dashboards.** Every project note IS its dashboard, with these
sections:

| Section | Embed |
|---|---|
| `## Tasks` | `![[All Tasks.base#Project Table]]` |
| `## Meetings` | `![[Meetings.base#Project Meetings]]` |
| `## Sessions` | `![[Sessions.base#Project Sessions]]` |
| `## Resources` | `![[Resources.base#Project Resources]]` |

Plus `## Overview`, `## Milestones`, and `## Key Decisions` as prose.
Initiative notes use the same dashboard with three differences:

- `## Tasks` embeds `![[All Tasks.base#Initiative Table]]`, grouped by Status
  (Completed, In Progress, Blocked, Not Started) with Discarded tasks hidden. It filters
  on `formula.Initiative == this`, so it lists the tasks filed straight under
  the initiative plus the tasks of every project whose `Initiative` points at
  it. It does not use `file.hasLink`, so a task that only mentions the
  initiative in its notes stays out. It lists the same tasks as the chat
  breakdown.
- `## Related Projects` sits between `## Milestones` and `## Meetings`. It holds
  `### Internal`, embedding `![[Projects.base#Initiative Internal Projects]]`,
  and `### Client`, embedding `![[Projects.base#Initiative Client Projects]]`.
  The split follows the project's folder, so it needs no property.
- `## Next Steps` follows `## Related Projects`. It lists what the user is waiting
  on that is not a task, one checkbox per line:
  `- [ ] <what> (waiting on [[Person]], since YYYY-MM-DD)`. The person is a
  wikilink, so the item backlinks from their note. Check a line off when it
  arrives. Delete it when it turns into a task. Write only the items the user
  gives; never infer one. Projects do not get this section.

Dashboards embed the **table** view,
not the kanban `Project Board`. The board still exists in `All Tasks.base` and
`Tasks/Tasks Kanban.base` for drag-drop work across every task at once.
`## Milestones` sits directly after `## Tasks` and holds a checkbox list, one
line per milestone, `- [ ] <name> — YYYY-MM-DD`. Dates are ISO in the note and
render as `Weekday, DD - Mon - YYYY` in chat breakdowns, where a concluded
milestone also gets a green ✅ after it.

**Chat breakdown of a project** runs in this order: Title, Target End Date,
Tasks (Concluded / In Progress / Blocked / Not Started, each bullet
`#<num> <name> — <date>`, ordered by due date with no-due-date last),
Milestones, then **Status**, which prints the `Status` frontmatter value verbatim
(`In Progress`, `Completed`, `Live Project`, `In Negotiation`, and so on). Never
map or simplify it. **Chat breakdown of an initiative** is the default
initiative view. It drops Target End Date and Status, since initiatives carry
neither, and runs in this order: Title, Tasks (Completed / In Progress /
Blocked / Not Started, each bullet `#<num> <name> (<date>)`, same ordering, Discarded left
out), Milestones, Related Projects split into Internal and Client by folder
(each `<name>: <Status>`, where that Status is the project's; an empty group
reads `none`), then Next Steps, open items only.

The `Project …` views live in
the **master bases** and filter on `file.hasLink(this.file)`, where `this` is the
embedding note. One definition serves every dashboard: edit it once and all
dashboards change. Opened directly in the master base, such a view shows empty.
That is expected. Nothing needs registering: any note whose frontmatter
wikilinks the project or initiative appears automatically, whichever property
holds the link.

**Wiki and Resources** (Karpathy LLM-wiki pattern) is a three-layer system,
additive to the CRM flows and isolated from them. `raw/` is the immutable Web
Clipper inbox. `Resources/` is the structured base. `wiki/` is LLM-owned
synthesis: topic folders with an `_index.md`, a front-door
`wiki/_master-index.md`, and an append-only `wiki/log.md`. The
`obsidian-process-resource` skill owns the procedure.

## Golden rules

**1. Frontmatter links are quoted wikilinks.**
Write `Company: "[[Acme]]"`. For multi-value, use a YAML list of `- "[[X]]"`.
Plain text does NOT create a backlink.

**Free-text values are always double-quoted.** Write `Notes: "…"` (and `Note:`
on Resources) in quotes every time, including when you append to an existing
value. Unquoted, a colon followed by a space, a leading `[[`, or a leading
quote breaks the YAML, and Bases silently drops the note. Escape inner double
quotes as `\"`. After editing frontmatter, check that it still parses.

**2. A wikilink target is the other note's basename.**
Basenames are folder-independent. Keep one note per entity. Reconcile duplicates
to the fullest name, and fix backlinks with an exact token replace.

**3. Folder layout.**

- People live in `<Company>` subfolders. Companies is flat.
- **`Projects/` splits by who the work is for**, one level above the company
  folders:
  - `Projects/Client projects/<Company>/`: one folder per client, as before.
  - `Projects/Internal projects/`: **flat**. Every internal project is your own
    company's, so a company folder inside it would carry no information.
- **Initiatives are flat in `Initiatives/`**, the way tasks are flat.
- **Tasks are flat in `Tasks/Task Notes/`.** No per-project subfolders. The base
  organizes them by the `Project` and `Initiative` properties.
- `Meetings/` has three subfolders:
  - `Inbox/`: the Granola sync dump. Transient and flat.
  - `Processed Meetings/` and `Transcripts/`: both bucketed into
    `<YYYY>-wk.<WW>/` ISO-week subfolders, for example `2026-wk.29/`. A note and
    its transcript always share a bucket. The base filter matches paths
    recursively, so the buckets stay indexed.
- `Tasks/` holds two more folders:
  - `Rundowns/`: digest notes tagged `Rundown`, never `Task`. They must stay out
    of the Tasks base and the kanban boards.
  - `Pending/`: holds the single **untagged** `Proposed Tasks.md` staging queue.
    A proposal is not a task: never tagged `Task`, never numbered, never on a
    board.

**4. Divisions are plain-text org units.**
A company lists its divisions. A person picks one, using the identical string.
Leave it blank for orgs with no division breakdown.

**Your own company's projects always carry `Division: internal` or
`Division: personal`.** Never leave a bare `Company: "[[Acme]]"` with no Division.
It separates the company's internal initiatives from your personal projects, so
the Projects base reads "Acme [internal]" and "Acme [personal]". This
project-level Division applies to your own company only, and differs from the
People division taxonomy (org units like Engineering and Sales). Client projects
do not use it. Set the company name in `Templates/Project.md` (`OWN_COMPANY`).

**5. Tasks, projects and initiatives.**

- `Task Number` is always quoted, as in `"1"`. Every task needs one.
- `Status` is one of: Not Started, In Progress, Blocked, Completed, Discarded.
- **`Blocked`** is an open status for a task that cannot move until something
  outside the user's control arrives. A Blocked task always carries `Blocker`, one
  quoted sentence that names what it waits on and who owns it, as in
  `Blocker: "Waiting on the PDF one-pager from [[Sarah Chen]]."` Write it
  only from what the user says. When the task moves to any other status, clear
  `Blocker` to blank. Blocked tasks are open: they appear on the rundown with
  their blocker sentence, but they are never booked into the calendar.
- No sub-task hierarchy. Every task is a flat note.
- A task has exactly one container, named in either `Project` or `Initiative`.
  `Project` is a quoted wikilink to a project note, or one of two plain-text
  labels:
  - `General task`: work with no project. Keep the normal `Company`.
  - `Personal Task`: the user's own to-dos, such as appointments and errands.
    Always pair it with `Company: N/A`, as plain text. `N/A` is used for nothing
    else.
- `Initiative` is a quoted wikilink to an initiative note. A task under an
  initiative leaves `Project` blank, and the reverse. **TaskNotes reads only
  `Project`** as its project field, so an initiative task shows a blank project
  in TaskNotes' own UI. Every Bases view still finds it, because the `Project …`
  views filter `file.hasLink(this.file)`, which matches a link in any property.
  Do not paper over this by filling both keys.
- **A project task belongs to its project's initiative too.** The Tasks base's
  **Initiative column is computed** (`formula.Initiative`): it shows the task's
  own `Initiative`, or else the `Initiative` of the task's project. The
  initiative dashboard's task table filters on that same formula, so a project
  task appears on both the project and the initiative dashboards. Never copy
  the initiative onto a project task: if the project moves to another
  initiative, a copied value goes stale and nothing flags it.
- `Priority` is an optional `High`, `Medium`, or `Low`. Blank means unset. It
  decides what gets booked first. It is never mandatory.
- `Effort` is the estimated minutes, always a multiple of 15. Leave it blank
  when unknown. Calendar slots are sized from it.
- `Booked ?` is a plain `Y` or `N`, defaulting to `N`. Flip it to `Y` once a
  calendar slot exists.
- **`Status` frontmatter is the single source of truth.** The kanban board
  (`Tasks/Tasks Kanban.base`, and the `Project Board` view for ad hoc use) is a
  TaskNotes view over it: it picks up tasks, and drag-drop writes `Status` back.
  **Never maintain board files by hand.**
- **Do not change the TaskNotes field mapping.** It is remapped to this vault's
  schema: tag `Task`, `Status`, `Due Date`, `Project`, `Created Date`.
- `Related Tasks` on a project or initiative lists its tasks as **aliased**
  wikilinks that display the number, `- "[[<task title>|#<num>]]"`. It is a
  hand-maintained index, so append to it whenever a task is created against that
  container. The embedded task table is the live view; `Related Tasks` is the
  at-a-glance list you can read in the frontmatter.
- **Projects** carry a sequential `Project Number`, always quoted. One global
  sequence across all companies. The Project template assigns (highest) + 1.
- **Initiatives** are open-ended containers, wider than a project, and carry
  **no Start or End Date, and no `Status`**. A project is finite and dated; an
  initiative runs until the user closes it, so a status field would only ever read
  "In Progress". The state that matters lives on its projects and tasks. They
  hold a sequential quoted `Initiative Number` on their own sequence, separate
  from `Project Number`. `Related Projects` links the projects that deliver
  pieces of the initiative, and each such project points back with
  `Initiative`.
- **Every project needs a target `End Date` and milestones.** When creating a
  project, ask the user for both: the target end date, and each milestone with its
  own end date. Suggest candidate milestones from the note's Overview, meetings,
  and open tasks, but **never write a milestone without his explicit
  authority.** A suggestion he did not answer creates nothing. Leaving
  `## Milestones` empty is correct until he decides; inventing entries is not.

**6. Never write a Task note off a meeting without the user's explicit sign-off.**
Filing the meeting is unconditional and immediate: the note, the transcript, the
entities, the `## Action items`. Creating tasks is not. Report the extraction,
then wait. Approval can be blanket, selective, or come with edits. Silence
creates nothing. Until approval, action-item bullets read
`(proposed task, pending validation)` as plain text, never as wikilinks. The
unattended auto-scan has nobody to ask, so it stages proposals in
`Tasks/Pending/Proposed Tasks.md`. This gate covers METHOD 2 only: a task the user
asks for directly is validated by the asking.
`obsidian-process-meeting` holds the reporting format.

**7. Never enrich a Granola note in place while it still has `granola_id`.**
The plugin owns any file carrying that key **anywhere in the vault**, and
re-indexes and clobbers it wherever it sits. Filing a note means renaming the
key to `granola_ref`. Moving the file is not enough.

**8. The Meetings base is folder-filtered on `Meetings/Processed Meetings/`,**
not tag-based. `Meetings/Inbox/` is deliberately excluded from Templater
folder-templates, because Granola auto-dumps there and a template would corrupt
the files. Deleting a raw inbox file does not stick while its meeting is inside
Granola's sync window, which re-dumps about every 30 minutes. That is expected.

**9. `raw/` is immutable.**
Read a clipping. Never edit, rename, move, or delete one. The meetings inbox is
different: that one is a move-out pipeline.

Tagging restrictions:

- `raw/` clippings and `wiki/` pages never carry `Task`, `Meeting`, or
  `Session`.
- `wiki/` pages additionally never carry `People`, `Company`, `Project`, or
  `Initiative`.
- Resource notes are the one exception: tagged `Resource`, in the Resources base.

**10. Editing care.**

- **Never edit `.base` view files** unless asked.
- **Prefer a blank property over an invented value.**
- After edits, **verify no broken wikilinks.**

**11. File every pasted document link.**
When the user pastes a Google Docs, Sheets, Slides, or Drive link in chat, store
the URL in the vault in the same turn, whatever else the request is about. Store
the link, not the document: read the content if the task needs it, but never
copy it into the note to "keep" it. The link is the retrieval path.

- **Where.** The most specific note the document belongs to: a task, then a
  project or initiative, then a company. A container-level document (SOW,
  scoping sheet, proposal, Drive folder) goes on the project, initiative, or
  company, even when a task prompted it. If no note fits, ask where it goes.
- **How.** A body section `## Links`, one bullet per document:
  `- [<doc title>](<url>): <what it is>. Added YYYY-MM-DD.` Take the title from
  the document when it is readable, otherwise from what the user said about it.
  Create the section at the end of the note if it is missing. Existing
  frontmatter `Link` and `Links` values stay as they are.
- **No duplicates.** Before adding, grep the vault for the document ID (the
  long token after `/d/` or `/folders/`). If the link is already filed on the
  right note, leave it. Drop the query string (everything from `?`), but keep
  a `#gid=` anchor, since it points at a specific Sheets tab.
- Say in the reply which note got the link.

## Templater folder templates

Templater (`Templates/`) scaffolds and auto-files new notes. Getting this map
wrong clobbers skill-generated notes.

| Folder | Template on create |
|---|---|
| `People/`, `Companies/`, `Projects/`, `Initiatives/`, `Tasks/` | Yes, the matching template. |
| `Meetings/Processed Meetings/` | Manual only (Insert Template), never folder-mapped. |
| `Tasks/Rundowns/`, `Tasks/Pending/` | **Shielded**: a no-op `Templates/Noop.md` plus guards in `Templates/Task.md`. |
| `Sessions/`, `raw/`, `Resources/`, `wiki/` | **None.** A skill (or the Web Clipper) creates these fully-formed. |

The `Projects/` mapping covers both subfolders, since Templater matches folder
prefixes. `Templates/Project.md` files a new note to
`Projects/Client projects/<Company>/`, or to `Projects/Internal projects/` when
the company is your own (`OWN_COMPANY`).

Session reports live with the session: a section in the session note, or a
companion note in `Sessions/<Project or Initiative>/`. There is **no top-level
Reports folder.**

**Fresh-vault order:** Companies first, with their Divisions. Then People,
Initiatives and Projects. Then Tasks.

## Rebuilding elsewhere

This repo *is* the empty scaffolding: clone it, open the folder as an Obsidian
vault, and start entering data. See `README.md` for the setup steps.

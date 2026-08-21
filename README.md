# Personal Assistant Vault (template)

A lightweight personal CRM + project/task system built on [Obsidian](https://obsidian.md),
designed to be driven by [Claude Code](https://claude.com/claude-code). It combines:

- **Obsidian Bases** (core plugin) — table views over plain Markdown notes:
  Companies, People, Projects, Tasks, Meetings, Sessions, Resources.
- **[Templater](https://github.com/SilentVoid13/Templater)** — auto-scaffolds new
  notes when you create them inside the right folder.
- **[TaskNotes](https://github.com/callumalpass/tasknotes)** — drag-drop kanban
  boards over task notes; dragging a card writes the task's `Status` back to its
  frontmatter.
- **Claude Code skills** (`.claude/skills/`) — teach Claude to create tasks from
  natural language, process meeting notes (e.g. from
  [Granola](https://granola.ai)) into crosslinked records + action items, book
  tasks into your calendar, log work sessions, turn web clippings into a wiki,
  keep everything crosslinked, and generate a daily task rundown.

Everything is plain Markdown with YAML frontmatter. No databases, no lock-in.

## Quick start

1. **Clone this repo** and open the folder as a vault in Obsidian
   (*Open folder as vault*).
2. Install and enable the **Templater** and **TaskNotes** community plugins,
   then reload Obsidian. Their configuration ships with the vault
   (`.obsidian/plugins/*/data.json`) and is picked up on load.
   - If Templater doesn't pick it up: set template folder to `Templates`, enable
     *Trigger Templater on new file creation*, and add folder templates
     People→`Templates/Person.md`, Companies→`Templates/Company.md`,
     Projects→`Templates/Project.md`, Tasks→`Templates/Task.md`,
     Tasks/Rundowns→`Templates/Noop.md`, Tasks/Pending→`Templates/Noop.md`.
3. Open `Templates/Project.md` and set `OWN_COMPANY` to your own company's note
   name (or `""` to switch off the internal/personal project split).
4. *(Optional)* Install a Granola→Obsidian sync plugin and point both notes and
   transcripts at `Meetings/Inbox/`; Claude's `obsidian-process-meeting` skill
   files them from there.
5. *(Optional)* Install the [Obsidian Web Clipper](https://obsidian.md/clipper)
   browser extension and set its save folder to `raw/`; the
   `obsidian-process-resource` skill turns clippings into `Resources/` notes and
   `wiki/` pages. See [raw/README.md](raw/README.md).
6. *(Optional)* Open the folder in **Claude Code** — `CLAUDE.md` and the skills
   under `.claude/skills/` teach it the whole system.
7. Start entering data: create a **Company** first (so People and Projects can
   link to it), then People, Projects, Tasks. Creating a note inside `People/`,
   `Companies/`, `Projects/`, or `Tasks/` triggers the matching template.

## How it works

| Base | Folder | Membership |
|---|---|---|
| Companies | `Companies/` | tagged `Company` |
| People | `People/<Company>/` | tagged `People` |
| Projects | `Projects/<Company>/` | tagged `Project` |
| Tasks | `Tasks/Task Notes/` (flat) | tagged `Task` |
| Meetings | `Meetings/Processed Meetings/` | folder-based |
| Sessions | `Sessions/<Project>/` | tagged `Session` |
| Resources | `Resources/` (flat) | tagged `Resource` |

- **Frontmatter wikilinks** (`Company: "[[Acme]]"`) wire everything together —
  every relation is a real backlink.
- **Every project note is its own dashboard**: it embeds filtered views of the
  master bases (tasks kanban, meetings, sessions, resources) that automatically
  show anything linking to the project. Nothing is registered anywhere.
- **Task `Status` frontmatter is the single source of truth**; the kanban boards
  are TaskNotes views over it. Tasks also carry `Priority`, `Effort` (minutes, in
  15-minute steps) and a `Booked ?` flag, which is what calendar booking reads.
- **Meetings flow**: raw notes land in `Meetings/Inbox/` (quarantined, transient),
  get processed into `Meetings/Processed Meetings/` (crosslinked, date-prefixed,
  bucketed by ISO week) with transcripts in `Meetings/Transcripts/`. Action items
  the user owns are **proposed** as tasks, and only become Task notes once the
  user approves them; unattended runs stage them in `Tasks/Pending/`.
- **Knowledge layer**: `raw/` is an immutable Web Clipper inbox, `Resources/` is
  the structured base over it, and `wiki/` holds LLM-owned synthesis pages
  (untagged, so they stay out of every base and board).

The full conventions live in [CLAUDE.md](CLAUDE.md); detailed procedures live in
the skills under [.claude/skills/](.claude/skills/).

## What Claude can do with it

- *"I need to send the proposal to Sarah by Friday"* → a crosslinked, due-dated
  task appears on the kanban.
- *"Process my meeting notes"* → inbox notes become crosslinked meeting records;
  new people/companies get notes automatically; your action items come back as
  proposals to approve.
- *"What's on my plate today?"* → a daily rundown note bucketed by due date.
- *"Book my tasks"* → open tasks are packed into free calendar slots, sized by
  their `Effort`, previewed before anything is created.
- *"Process my resources"* → web clippings become Resource notes and wiki pages.
- *"I'm done for today"* → a session log with what was done, decisions made, and
  task statuses updated.

## Calendar booking

`obsidian-book-tasks` needs a calendar tool wired into Claude Code (any MCP
calendar server with `list_events` / `create_event`). The skill assumes a daily
`Work Block` event and falls back to any free gap in the working day. Set your
own calendar address at the top of
[.claude/skills/obsidian-book-tasks/SKILL.md](.claude/skills/obsidian-book-tasks/SKILL.md)
(it ships as `you@example.com`).

## License

MIT — use it, fork it, share it.

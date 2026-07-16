# Personal Assistant Vault (template)

A lightweight personal CRM + project/task system built on [Obsidian](https://obsidian.md),
designed to be driven by [Claude Code](https://claude.com/claude-code). It combines:

- **Obsidian Bases** (core plugin) — table views over plain Markdown notes:
  Companies, People, Projects, Tasks, Meetings, Sessions.
- **[Templater](https://github.com/SilentVoid13/Templater)** — auto-scaffolds new
  notes when you create them inside the right folder.
- **[TaskNotes](https://github.com/callumalpass/tasknotes)** — drag-drop kanban
  boards over task notes; dragging a card writes the task's `Status` back to its
  frontmatter.
- **Claude Code skills** (`.claude/skills/`) — teach Claude to create tasks from
  natural language, process meeting notes (e.g. from
  [Granola](https://granola.ai)) into crosslinked records + action items, log
  work sessions, keep everything crosslinked, and generate a daily task rundown.

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
     Tasks/Rundowns→`Templates/Noop.md`.
3. *(Optional)* Install a Granola→Obsidian sync plugin and point both notes and
   transcripts at `Meetings/Inbox/`; Claude's `obsidian-process-meeting` skill
   files them from there.
4. *(Optional)* Open the folder in **Claude Code** — `CLAUDE.md` and the skills
   under `.claude/skills/` teach it the whole system.
5. Start entering data: create a **Company** first (so People and Projects can
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

- **Frontmatter wikilinks** (`Company: "[[Acme]]"`) wire everything together —
  every relation is a real backlink.
- **Every project note is its own dashboard**: it embeds filtered views of the
  master bases (tasks kanban, meetings, sessions) that automatically show
  anything linking to the project. Nothing is registered anywhere.
- **Task `Status` frontmatter is the single source of truth**; the kanban boards
  are TaskNotes views over it.
- **Meetings flow**: raw notes land in `Meetings/Inbox/` (quarantined, transient),
  get processed into `Meetings/Processed Meetings/` (crosslinked, date-prefixed)
  with transcripts in `Meetings/Transcripts/`, and the user's action items become
  Task notes.

The full conventions live in [CLAUDE.md](CLAUDE.md); detailed procedures live in
the skills under [.claude/skills/](.claude/skills/).

## What Claude can do with it

- *"I need to send the proposal to Sarah by Friday"* → a crosslinked, due-dated
  task appears on the kanban.
- *"Process my meeting notes"* → inbox notes become crosslinked meeting records;
  new people/companies get notes automatically; your action items become tasks.
- *"What's on my plate today?"* → a daily rundown note bucketed by due date.
- *"I'm done for today"* → a session log with what was done, decisions made, and
  task statuses updated.

## License

MIT — use it, fork it, share it.

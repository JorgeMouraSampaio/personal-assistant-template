# Proposed Tasks (pending validation)

Staging queue for tasks mined from processed meetings that the user has not yet
validated. **Nothing in this file is a Task.** The note is deliberately untagged,
so it stays out of the Tasks base, the kanban boards, and the daily rundown.

Written by the unattended meeting auto-scan (`obsidian-process-meetings`), read
back for approval by `obsidian-process-meeting` and flagged in the daily rundown.
On approval the proposal becomes a real Task note in `Tasks/Task Notes/` and its
block is deleted from here. Rejected proposals are deleted too, with a one-line
note in the meeting's `## Action items`.

One `###` block per meeting, one `-` bullet per proposed task. No `Task Number` is
stored: numbers are projected at read-back time and assigned for real at creation.

<!-- proposals below this line; when there are none, the file ends here -->

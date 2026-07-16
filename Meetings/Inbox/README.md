---
tags:
  - GranolaInbox
---
# Granola inbox (transient — do not edit)

The Granola Sync plugin dumps raw meeting notes and transcripts here on every sync.
These files are **transient**: they carry Granola's `granola_id` and are overwritten
/ re-created on each sync. The `obsidian-process-meeting` skill moves each one into
`Meetings/Processed Meetings/` (clean note) and `Meetings/Transcripts/` (transcript),
renaming `granola_id` → `granola_ref` so the filed copies are permanent.

Nothing here is authoritative. Ignore this folder in day-to-day use.

---
tags:
  - RawInbox
---
# Raw inbox (immutable — read, never modify)

This folder is the **source of truth** for external resources. The Obsidian **Web
Clipper** (Chrome/browser extension) saves clippings here: articles, posts,
blogposts, news, internal docs. Following Karpathy's LLM-wiki pattern, these files
are **immutable** — Claude reads from them but **never edits, renames, moves, or
deletes them**. If a source needs correcting, fix it upstream and re-clip.

## What the clipper writes

Filename: `YYYY-MM-DD-<Title>.md` (clip date prefix). Default *Clippings* template
frontmatter:

```yaml
---
title: "..."
source: "https://..."
author:
  - "[[Name]]"        # or blank
published: 2026-06-02  # or blank
created: 2026-06-25    # clip date
description: "..."
tags:
  - "clippings"
---
```

## What happens next

The `obsidian-process-resource` skill (manual: "process my resources" / "compile",
or the scheduled auto-scan) reads each clipping and, **without touching it**,
produces a structured note in `Resources/` (the Resources base) and weaves the
takeaways into `wiki/`. Authors become People notes, source-orgs become Company
notes, and a `Raw` back-link on the resource note points back here.

Nothing here is authoritative on its own — it is raw input. Ignore this folder in
day-to-day use.

## Google Docs don't clip

The Web Clipper cannot read the live Google Docs editor
(`docs.google.com/document/.../edit`) — it saves only page UI chrome (iframes,
outline, account info), not the document text. Two workarounds:

1. **Paste the doc text into chat** and ask to "process my resources" — Claude merges
   the real text into this clipping (keeping the source URL / frontmatter) and
   recompiles into `Resources/` + `wiki/`. This is the one time a `raw/` file is
   rewritten, and only to make the source truthful.
2. **Download it** from Google Docs (File → Download → Markdown or PDF) or clip a
   **published** version — both capture cleanly.

## Web Clipper setup (one-time, done in the browser)

In the Obsidian Web Clipper extension settings: set **Vault** to this vault and
**Note location / folder** to `raw`, and keep the default *Clippings* template.
That is the only browser-side step; everything after is handled in the vault.

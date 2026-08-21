---
name: obsidian-process-resource
description: >-
  Ingest raw resources (articles, posts, blogposts, news, internal docs) clipped
  by the Obsidian Web Clipper into this vault's raw/ inbox, turning each into a
  structured Resource note (the Resources base) and weaving its takeaways into the
  LLM-owned wiki/. Use whenever the user says "process my resources", "compile",
  "ingest this", "add this to the wiki", "process the raw folder", or points at a
  note in raw/. Reads raw/ but NEVER modifies it (immutable source of truth).
  Writes a Resource note (Title, Link, Type, Author, Company, Project, Date, Note)
  in Resources/, auto-creates missing People (authors) / Company (source-orgs) /
  Project notes and crosslinks them, then creates/updates wiki topic pages,
  indexes, and the log. Also handles querying and auditing (linting) the wiki.
---

# Process Resource (raw/ clipping → Resource note + wiki pages)

The Obsidian **Web Clipper** dumps external sources into `raw/` as Markdown with
the clipper's default *Clippings* frontmatter. This skill turns one raw clipping
into (a) a **structured Resource note** in `Resources/` (the Resources base), wired
into the People/Companies/Projects graph, and (b) **wiki synthesis pages** in
`wiki/` (Karpathy's LLM-owned layer). It also covers **querying** and **auditing**
the wiki.

This is the third task-entry / ingest method in the vault, alongside
`obsidian-create-task` (METHOD 1) and `obsidian-process-meeting` (METHOD 2).

**`raw/` is immutable — read, never write.** Following Karpathy's LLM-wiki pattern,
raw clippings are the source of truth: never edit, rename, move, or delete a file
in `raw/`. All output goes to `Resources/` and `wiki/`. (This is the opposite of
the meetings inbox, which is a move-out pipeline — here the raw file stays put.)

Read `obsidian-base-cleanup` (vault model, wikilink quoting, `People/<Company>/`
folders, entity creation, broken-link check) first — this skill composes it.

## Three-layer model

| Layer | Folder | Owner | Mutability |
|---|---|---|---|
| Raw sources | `raw/` | Web Clipper | **Immutable** — read only |
| Structured | `Resources/` | this skill | one `Resource`-tagged note per source = the Resources base |
| Synthesis | `wiki/` | this skill | topic pages + `_master-index.md` + `log.md`; **untagged**, so never in a Base/board |

`wiki/` pages carry **no tags and no frontmatter** (except a topic `_index.md`'s
single `Status:` prop) — that is what keeps them out of every Base, kanban, and the
daily rundown. Never add `Task`, `Meeting`, `Session`, `People`, `Company`, or
`Project` tags to a wiki or raw file.

## Resource note schema

Filed at `Resources/<Sanitized Title>.md`, tagged `Resource`:

```yaml
---
tags:
  - Resource
Title: "How agentic retrieval changes internal search"
Link: https://example.com/articles/agentic-retrieval
Type: article            # one of: post | article | blogpost | news | internal doc
Author:
  - "[[Sarah Chen]]"
Company:                 # blank, or "[[Acme Labs]]" when a source-org applies
Project:                 # blank, or "[[<Project>]]"
Date: 2026-06-02         # published date (clipper `published`); fall back to `created`
Note: One-sentence summary of what this source says.
Raw: "[[2026-06-25-How agentic retrieval changes internal search]]"
---
```

- **Golden rules apply**: links are quoted wikilinks; multi-value (e.g. multiple
  authors) is a YAML list of `- "[[X]]"`; wikilink target is the other note's
  basename. Prefer a blank property over an invented value.
- **`Type`** is exactly one of `post | article | blogpost | news | internal doc`.
- **`Raw`** back-links the immutable clipping (its basename, including the date
  prefix). The wiki and the graph point at the Resource note, never at `raw/`.
- **Body**: a short synthesis is optional; the canonical summary lives in `Note`
  and the depth lives in the wiki. Do NOT paste the raw article body into the
  Resource note (it stays in `raw/`).

## Workflow (ingest / "compile")

### 1. Read the raw clipping(s) — never modify raw/
Read the target file in `raw/` (the user names one, or process every clipping not
yet ingested; the auto-scan does all unfiled ones). Skip `README.md`.
**Gate on a usable body.** Google Docs and PDF sources often clip as UI chrome or
an empty body (only an `<iframe>`, "Drag image to reposition", or a bare PDF URL).
If the body has no real content, do **not** invent it: create no wiki article, and
record the source in the topic's provenance table with a ⚠️ note ("Google Doc —
body didn't capture" / "PDF link only"). You may still create a thin Resource note
from the frontmatter if the title/source are meaningful, marking `Note` as
"⚠️ body not captured".
**Google Docs workaround.** The Web Clipper cannot read the live Google Docs editor
(`docs.google.com/document/.../edit`) — it only ever captures UI chrome. To ingest
one properly, the user supplies the real text one of two ways: (a) **paste the doc
body into chat** and you merge it into the existing `raw/` clipping (keep the
clipper frontmatter — source URL, tags, `created` — and replace only the junk body),
then recompile; or (b) re-clip a **File → Download → Markdown/PDF** export or a
*published* version, which capture cleanly. Merging supplied text into `raw/` is the
one sanctioned exception to "never write to raw/", because it makes the source of
truth actually true; note it in the log (`recompile:` line) and clear the ⚠️.

### 2. Dedup
A clipping is already ingested if an existing `Resources/` note has the same
`Link` (its `source` URL) or already back-links it via `Raw`. If so, skip it
(optionally refresh the wiki if the source changed). This is what makes the
scheduled auto-scan idempotent — `raw/` is never emptied, so dedup is by content,
not by folder.

### 3. Parse the clipper frontmatter and infer Type
From the clipping's frontmatter read: `title` → **Title**, `source` → **Link**,
`author` → **Author** (may already be a `[[wikilink]]`; may be blank), `published`
(else `created`) → **Date**, `description` → seed for **Note**. Draft a **one-sentence
`Note`** from the body/description.
**Infer `Type`**: news outlet → `news`; journal / research / long-form explainer →
`article`; personal site / Substack / company engineering blog → `blogpost`;
X/Twitter / LinkedIn / Mastodon / short social → `post`; a doc from your own
org/Drive/Notion → `internal doc`. When ambiguous, prefer `article`.

### 4. Resolve / create graph entities (always register) — per obsidian-base-cleanup
- **Author → People note.** Match an existing person by basename first; else
  **create** one tagged `[People]`, filed `People/<Org>/` when the org is known
  (or already a Company note), otherwise `People/External/` with
  `Company: External` (plain text). Fill only what the source honestly tells you;
  leave Role/Email/etc. blank. Add a `Notes:` line noting they authored this
  source, e.g. `Author of [[<Resource>]].`
- **Source-org → Company note.** If the source is published by an org that should
  show up as a company (e.g. a post from `Acme Labs` → `Acme Labs`), match an existing
  `Company` note first, else **create** one tagged `[Company]`. If the org is
  already a tracked CRM company (a client/vendor), just link it. Group labels
  ("the research team") are not companies — skip them.
- **Project → link** only if the source clearly ties to a tracked project; else
  leave `Project` blank.
- Authors and source-orgs are **normal People/Company notes** and appear in the
  normal People/Companies bases by design (no separate tag or view).

### 5. Write the Resource note
Create `Resources/<Sanitized Title>.md` with the schema above. Sanitize the
filename (drop characters Obsidian dislikes: `/ : # ^ [ ] |`; keep it readable).
Set `Raw` to the clipping's basename. This note is the graph hub: its
`Author`/`Company`/`Project` wikilinks backlink onto those notes automatically.

### 6. Weave into the wiki ("compile")
This is the Karpathy step — synthesize, don't just file.
- **Pick or create a topic.** Topics are folders `wiki/<lowercase-hyphen-topic>/`.
  Reuse an existing topic when the source fits; create a new one when it doesn't.
  the user is intentionally un-opinionated about structure — group by subject the way
  a human librarian would.
- **Write/update article note(s)** `wiki/<topic>/<lowercase-hyphen-title>.md`
  (no frontmatter): `# Title`, bullet-heavy synthesis, inline `[[wikilinks]]` to
  related articles and to the `[[Resource]]` note, ending in a `## Key Takeaways`
  section. If the source informs an existing article, **update it** rather than
  duplicating; a single source can touch several articles.
- **Topic `_index.md`** (`wiki/<topic>/_index.md`) — frontmatter is a single
  `Status:` prop; body has `# <Topic>`, a one-paragraph definition, `## Articles`
  (bulleted `[[article]] — description`), `## Related Topics` (links to sibling
  topics, which may not exist yet), and a `## Resources processed` provenance
  table:

  ```markdown
  ## Resources processed
  | Resource | Status |
  |---|---|
  | [[<Resource note>]] | ✅ Compiled → what-are-virtual-cells, perturbation-prediction |
  | [[<Resource note>]] | ⚠️ Google Doc — body didn't capture |
  ```
- **Update `wiki/_master-index.md`** — add/refresh the topic's row in the `## Topics`
  table (`| [[<topic>/\|Topic Name]] | one-line summary |`).
- **Append to `wiki/log.md`** — one dated line, e.g.
  `- 2026-07-16 ingest: "<Title>" → resource + wiki/<topic>/<article> (+ index)`.

### 7. Verify & report
Run the broken-wikilink check from `obsidian-base-cleanup`: resource→author,
resource→company/project, resource→raw, wiki article↔resource, topic↔master-index
must all resolve. `raw/` must be **byte-for-byte unchanged**. Then summarize:
which clippings were ingested, the Resource notes created, People/Companies created
vs. linked, which wiki topics/articles were written or updated, and any ⚠️
empty-capture sources.

## Querying the wiki
When the user asks a question against their knowledge base ("what do my notes say
about X", "query the wiki"):
1. Read `wiki/_master-index.md` → the relevant topic `_index.md` → the articles.
2. Synthesize an answer **with citations** to the wiki articles and the underlying
   `[[Resource]]` notes (and their `Link`).
3. If the answer is durably useful, offer to **file it back** as a new wiki article
   (Karpathy's "good answers become pages") and log it in `wiki/log.md`.

## Auditing the wiki (lint)
When asked to "audit"/"lint"/"health-check" the wiki: walk `wiki/` and report (do
**not** edit) — broken wikilinks, orphan articles (not linked from any `_index.md`),
topics missing from `_master-index.md`, `Resources/` notes never woven into the
wiki, contradictions or stale claims across articles, and coverage gaps. Output a
findings list; make changes only when the user then asks.

## Judgment & safety
- **`raw/` is immutable.** Never edit, rename, move, or delete a clipping. If a
  capture is bad, note it and move on — don't reconstruct content you can't see.
- **Don't fabricate.** Prefer a blank property, a ⚠️ provenance note, or "not
  captured" over invented authors, dates, or summaries.
- **Always register** authors and identifiable source-orgs (People/Company notes),
  matching existing notes by basename before creating — never duplicate an entity.
- **Keep wiki/raw untagged** with the CRM tags — a stray `Task`/`Meeting`/etc. tag
  would leak these notes into the wrong Base or the daily rundown.
- **Never touch** the Tasks/Meetings/Sessions bases, the TaskNotes mapping, or the
  meeting/rundown pipelines. This skill is additive and isolated.
- Verify no broken wikilinks before reporting done.

<%*
// New project: pick its company, set Company as a wikilink, seed dates,
// and file the note under Projects/<Company>/. The body is the project
// DASHBOARD: overview + the "Project …" views of the master Tasks/Meetings/
// Sessions bases, embedded by reference. Those views filter on `this.file`
// (= this project note), so anything whose frontmatter links to the project
// shows up automatically — and the layout is maintained centrally in the
// master .base files, not per note.
const companyFiles = app.vault.getMarkdownFiles()
  .filter(f => f.path.startsWith("Companies/") && f.basename !== "Company Base");
const names = companyFiles.map(f => f.basename).sort();
const company = await tp.system.suggester(names, names, false, "Company?");
if (!company) return;

const status = (await tp.system.prompt("Status? (optional)", "", false)) ?? "";
const link = (await tp.system.prompt("Link? (optional)", "", false)) ?? "";
const start = tp.date.now("YYYY-MM-DD");
const end = tp.date.now("YYYY-MM-DD", 90);

let name = tp.file.title;
if (name.startsWith("Untitled")) name = (await tp.system.prompt("Project title?", "", false)) || name;

tR += `---
tags:
  - Project
Company: "[[${company}]]"
Start Date: ${start}
End Date: ${end}
Status: ${status}
Link: ${link}
---
## Overview


## Tasks

![[All Tasks.base#Project Board]]

## Meetings

![[Meetings.base#Project Meetings]]

## Sessions

![[Sessions.base#Project Sessions]]

## Key Decisions

`;

const folder = "Projects/" + company;
if (!app.vault.getAbstractFileByPath(folder)) await app.vault.createFolder(folder);
await tp.file.move(folder + "/" + name);
%>

<%*
// New project: pick its company, set Company as a wikilink, seed dates,
// and file the note under Projects/Client projects/<Company>/ (or
// Projects/Internal projects/ when the company is your own). The body is the
// project DASHBOARD: overview + the "Project …" views of the master Tasks/
// Meetings/Sessions/Resources bases, embedded by reference. Those views filter
// on `this.file` (= this project note), so anything whose frontmatter links to
// the project shows up automatically — and the layout is maintained centrally
// in the master .base files, not per note.
const companyFiles = app.vault.getMarkdownFiles()
  .filter(f => f.path.startsWith("Companies/") && f.basename !== "Company Base");
const names = companyFiles.map(f => f.basename).sort();
const company = await tp.system.suggester(names, names, false, "Company?");
if (!company) return;

// Next project number = (highest existing number) + 1
let maxNum = 0;
for (const f of app.vault.getMarkdownFiles().filter(f => f.path.startsWith("Projects/"))) {
  const pn = app.metadataCache.getFileCache(f)?.frontmatter?.["Project Number"];
  if (pn != null) {
    const n = parseInt(String(pn), 10);
    if (!isNaN(n) && n > maxNum) maxNum = n;
  }
}
const number = `${maxNum + 1}`;

// Your OWN company's projects are classified internal vs personal (see
// CLAUDE.md); other companies' projects carry no Division. Set OWN_COMPANY to
// the basename of your own company's note, or "" to switch this off.
const OWN_COMPANY = "Acme";
let division = "";
if (OWN_COMPANY && company === OWN_COMPANY) {
  division = await tp.system.suggester(["internal", "personal"], ["internal", "personal"], false, "Project type?");
  if (!division) return;
}
const divisionLine = division ? `Division: ${division}\n` : "";

// A project may sit under an initiative (open-ended container). Optional.
const initFiles = app.vault.getMarkdownFiles().filter(f => f.path.startsWith("Initiatives/"));
const initNames = initFiles.map(f => f.basename).sort();
const initiative = initNames.length
  ? (await tp.system.suggester(["— none —", ...initNames], ["", ...initNames], false, "Initiative? (optional)")) ?? ""
  : "";

const status = (await tp.system.prompt("Status? (optional)", "", false)) ?? "";
const link = (await tp.system.prompt("Link? (optional)", "", false)) ?? "";
const start = tp.date.now("YYYY-MM-DD");
// Every project needs a TARGET end date. Ask for it rather than inventing one;
// the +90d value is only a pre-filled suggestion, and blank is better than wrong.
const end = (await tp.system.prompt("Target end date? (YYYY-MM-DD)", tp.date.now("YYYY-MM-DD", 90), false)) ?? "";

let name = tp.file.title;
if (name.startsWith("Untitled")) name = (await tp.system.prompt("Project title?", "", false)) || name;

tR += `---
tags:
  - Project
Project Number: "${number}"
Company: "[[${company}]]"
${divisionLine}Initiative: ${initiative ? `"[[${initiative}]]"` : ""}
Start Date: ${start}
End Date: ${end}
Status: ${status}
Stakeholders:
Related Tasks:
Link: ${link}
Slack Channels:
Slack DMs:
---
## Overview


## Tasks

![[All Tasks.base#Project Table]]

## Milestones

## Meetings

![[Meetings.base#Project Meetings]]

## Sessions

![[Sessions.base#Project Sessions]]

## Resources

![[Resources.base#Project Resources]]

## Key Decisions

`;

const folder = (OWN_COMPANY && company === OWN_COMPANY)
  ? "Projects/Internal projects"
  : "Projects/Client projects/" + company;
if (!app.vault.getAbstractFileByPath(folder)) await app.vault.createFolder(folder);
await tp.file.move(folder + "/" + name);
%>

<%*
// New initiative: an open-ended container, wider than a project and with no
// start or end date, and no Status. It groups related projects and tasks under
// one heading.
// Initiative Number auto-increments on its own sequence, separate from the
// project sequence. The body is the initiative DASHBOARD, built from the same
// central "Project …" base views the project dashboards use: they filter on
// `this.file`, so anything whose frontmatter links to this note shows up.
const companyFiles = app.vault.getMarkdownFiles()
  .filter(f => f.path.startsWith("Companies/") && f.basename !== "Company Base");
const names = companyFiles.map(f => f.basename).sort();
const company = await tp.system.suggester(names, names, false, "Company?");
if (!company) return;

// Next initiative number = (highest existing number) + 1
let maxNum = 0;
for (const f of app.vault.getMarkdownFiles().filter(f => f.path.startsWith("Initiatives/"))) {
  const n0 = app.metadataCache.getFileCache(f)?.frontmatter?.["Initiative Number"];
  if (n0 != null) {
    const n = parseInt(String(n0), 10);
    if (!isNaN(n) && n > maxNum) maxNum = n;
  }
}
const number = `${maxNum + 1}`;

const link = (await tp.system.prompt("Link? (optional)", "", false)) ?? "";

let name = tp.file.title;
if (name.startsWith("Untitled")) name = (await tp.system.prompt("Initiative title?", "", false)) || name;

tR += `---
tags:
  - Initiative
Initiative Number: "${number}"
Company: "[[${company}]]"
Stakeholders:
Related Projects:
Related Tasks:
Link: ${link}
Slack Channels:
Slack DMs:
---
## Overview


## Tasks

![[All Tasks.base#Initiative Table]]

## Milestones

## Related Projects

### Internal

![[Projects.base#Initiative Internal Projects]]

### Client

![[Projects.base#Initiative Client Projects]]

## Next Steps

## Meetings

![[Meetings.base#Project Meetings]]

## Sessions

![[Sessions.base#Project Sessions]]

## Resources

![[Resources.base#Project Resources]]

## Key Decisions

`;

// Initiatives are flat in Initiatives/, the way tasks are flat in Tasks/.
const folder = "Initiatives";
if (!app.vault.getAbstractFileByPath(folder)) await app.vault.createFolder(folder);
await tp.file.move(folder + "/" + name);
%>

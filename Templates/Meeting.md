<%*
// Manual meeting note. (Granola auto-dumps raw notes/transcripts into the
// Meetings/ inbox — those do NOT go through this template; they're filed into
// "Processed Meetings/" by the obsidian-process-meeting skill.) Prompts for an
// optional company/project, seeds today's date, and files the note into the
// "Processed Meetings/" folder that backs the Meetings base.
const companyFiles = app.vault.getMarkdownFiles()
  .filter(f => f.path.startsWith("Companies/"));
const cNames = companyFiles.map(f => f.basename).sort();
const company = (await tp.system.suggester(["— none —", ...cNames], ["", ...cNames], false, "Company? (optional)")) ?? "";

const projFiles = app.vault.getMarkdownFiles().filter(f => f.path.startsWith("Projects/"));
const pNames = projFiles.map(f => f.basename).sort();
const project = (await tp.system.suggester(["— none —", ...pNames], ["", ...pNames], false, "Project? (optional)")) ?? "";

const date = tp.date.now("YYYY-MM-DD");
const prefix = tp.date.now("YYYYMMDD");   // meeting-date prefix so the Meetings base reads chronologically
let name = tp.file.title;
if (name.startsWith("Untitled")) name = (await tp.system.prompt("Meeting title?", "", false)) || name;

tR += `---
tags:
  - Meeting
Date: ${date}
Attendees:
Company: ${company ? `"[[${company}]]"` : ""}
Project: ${project ? `"[[${project}]]"` : ""}
Links:
---
# ${name}

## Attendees

## Notes

## Action items
`;
const folder = "Meetings/Processed Meetings";
if (!app.vault.getAbstractFileByPath(folder)) await app.vault.createFolder(folder);
await tp.file.move(folder + "/" + prefix + " " + name);
%>

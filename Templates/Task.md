<%*
// New task. Pick a project (or "General task"); company is inherited
// from the project. Task Number auto-increments to the next whole number.
// Rundown notes and the pending-proposal queue are generated fully-formed:
// never template them.
if (tp.file.path(true).startsWith("Tasks/Rundowns/")) return;
if (tp.file.path(true).startsWith("Tasks/Pending/")) return;
const projFiles = app.vault.getMarkdownFiles()
  .filter(f => f.path.startsWith("Projects/"));
const projNames = projFiles.map(f => f.basename).sort();
const project = await tp.system.suggester(
  ["— General task —", ...projNames], ["General task", ...projNames], false, "Project?");
if (project === null) return;

let company = "";
if (project !== "General task") {
  const pFile = projFiles.find(f => f.basename === project);
  const cv = pFile ? app.metadataCache.getFileCache(pFile)?.frontmatter?.Company : null;
  if (cv) company = String(cv).replace(/\[\[|\]\]/g, "").trim();
}

// Next task number = (highest existing number) + 1
let maxMain = 0;
for (const f of app.vault.getMarkdownFiles().filter(f => f.path.startsWith("Tasks/"))) {
  const tn = app.metadataCache.getFileCache(f)?.frontmatter?.["Task Number"];
  if (tn != null) {
    const n = parseInt(String(tn).split(".")[0], 10);
    if (!isNaN(n) && n > maxMain) maxMain = n;
  }
}
const number = `${maxMain + 1}`;

const created = tp.date.now("YYYY-MM-DD");
const due = (await tp.system.prompt("Due date? YYYY-MM-DD (optional)", "", false)) ?? "";
// Priority is optional (blank = unset); used to decide what to book first.
const priority = (await tp.system.suggester(
  ["— none —", "High", "Medium", "Low"], ["", "High", "Medium", "Low"], false, "Priority? (optional)")) ?? "";
// Effort in minutes, multiples of 15 (used to book calendar slots). Optional.
const effort = (await tp.system.prompt("Effort? minutes, multiples of 15 (optional)", "", false)) ?? "";
const link = (await tp.system.prompt("Link? (optional)", "", false)) ?? "";

let name = tp.file.title;
if (name.startsWith("Untitled")) name = (await tp.system.prompt("Task title?", "", false)) || name;

tR += `---
tags:
  - Task
Task Number: "${number}"
Status: Not Started
Project: ${project === "General task" ? "General task" : `"[[${project}]]"`}
Stakeholders:
Company: ${company ? `"[[${company}]]"` : ""}
Created Date: ${created}
Due Date: ${due}
Priority: ${priority}
Effort: ${effort}
Booked ?: N
Notes:
Links: ${link}
---
`;
// All task notes live flat in Tasks/Task Notes/; the Project property (not the
// folder) is what the Tasks base organizes by.
const folder = "Tasks/Task Notes";
if (!app.vault.getAbstractFileByPath(folder)) await app.vault.createFolder(folder);
await tp.file.move(folder + "/" + name);
%>

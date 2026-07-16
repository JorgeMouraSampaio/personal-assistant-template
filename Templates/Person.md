<%*
// New person: pick their company, then a division from THAT company's list,
// set Company as a wikilink, and file the note under People/<Company>/.
const companyFiles = app.vault.getMarkdownFiles()
  .filter(f => f.path.startsWith("Companies/") && f.basename !== "Company Base");
const names = companyFiles.map(f => f.basename).sort();
const company = await tp.system.suggester(names, names, false, "Company?");
if (!company) return;

// Division options come from the chosen company's Division property.
let divisions = [];
const cFile = companyFiles.find(f => f.basename === company);
if (cFile) {
  const dv = app.metadataCache.getFileCache(cFile)?.frontmatter?.Division;
  if (Array.isArray(dv)) divisions = dv.filter(Boolean);
  else if (dv) divisions = [dv];
}
let division = "";
if (divisions.length) {
  division = await tp.system.suggester([...divisions, "— none —"], [...divisions, ""], false, "Division?") ?? "";
}

const role = (await tp.system.prompt("Role?", "", false)) ?? "";
const email = (await tp.system.prompt("Email? (optional)", "", false)) ?? "";
const linkedin = (await tp.system.prompt("LinkedIn? (optional)", "", false)) ?? "";
const notes = (await tp.system.prompt("Notes? (optional)", "", false)) ?? "";

let name = tp.file.title;
if (name.startsWith("Untitled")) name = (await tp.system.prompt("Name?", "", false)) || name;

tR += `---
tags:
  - People
Company: "[[${company}]]"
Division: ${division}
Role: ${role}
Email: ${email}
LinkedIn: ${linkedin}
Notes: ${notes}
---
`;

const folder = "People/" + company;
if (!app.vault.getAbstractFileByPath(folder)) await app.vault.createFolder(folder);
await tp.file.move(folder + "/" + name);
%>

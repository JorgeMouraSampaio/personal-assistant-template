<%*
// New company: every org (client, vendor, consultancy, internal) is tagged
// `Company` and appears in the Companies base. Optionally list its divisions.
const divRaw = (await tp.system.prompt("Divisions? comma-separated (optional)", "", false)) ?? "";
const divisions = divRaw ? divRaw.split(",").map(s => s.trim()).filter(Boolean) : [];
const divBlock = divisions.length
  ? "Division:\n" + divisions.map(d => `  - ${d}`).join("\n") + "\n"
  : "Division: \n";

let name = tp.file.title;
if (name.startsWith("Untitled")) name = (await tp.system.prompt("Company name?", "", false)) || name;

tR += `---
tags:
  - Company
${divBlock}Stakeholders:
Projects:
Meetings:
Notes:
Slack Channels:
Slack DMs:
---
`;
await tp.file.move("Companies/" + name);
%>

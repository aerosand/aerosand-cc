<%-*
/* ---------- 工具函数 ---------- */
async function sanitizeName(name) {
  return name.trim().replace(/[\\\/:*?"<>|]/g, " ").replace(/\s+/g, " ").trim();
}
async function ensureFolder(folder) {
  if (!folder) return "";
  const clean = folder.replace(/\/+$/,"").trim();
  if (!clean) return "";
  const exists = await app.vault.adapter.exists(clean);
  if (!exists) await app.vault.createFolder(clean);
  return clean;
}
async function uniquePath(path) {
  let p = path; let i = 1;
  while (await app.vault.adapter.exists(p + ".md")) {   // 只判断 .md 文件是否存在
    p = `${path} (${i++})`;
  }
  return p;
}

/* ---------- 1. 获取有效文件名 ---------- */
let title = tp.file.title || "";
const invalid = ["untitled","未命名","新文档"];
const isInvalid = !title || invalid.some(v => title.toLowerCase().includes(v.toLowerCase()));
if (isInvalid) {
  const input = await tp.system.prompt("Input note name：");
  title = (input && input.trim()) ? input.trim() : `Note_${new Date().toISOString().replace(/[:.-]/g,"").slice(0,14)}`;
}
title = await sanitizeName(title);

/* ---------- 2. 文件夹选择 ---------- */
let destFolder = "";
const firstChoice = await tp.system.suggester(
  ["ofs","news","/"],
  ["ofs","news","__ROOT__"],
  false,
  "Please select folder"
);

if (firstChoice === "ofs") {
  const inputFolder = await tp.system.prompt("Input subfolder（Blank for [ofs/] ）");
  destFolder = inputFolder && inputFolder.trim() ? `Project/${inputFolder.trim()}` : "Project";
} else if (firstChoice === "news") {
  const inputFolder = await tp.system.prompt("Input subfolder（Blank for [news/] ）");
  destFolder = inputFolder && inputFolder.trim() ? `Self/${inputFolder.trim()}` : "Self";
} else {
  destFolder = ""; // 根目录
}

/* ---------- 3. 移动 + 改名 ---------- */
let finalPath = title; // 注意：不加 .md，交给 Obsidian 自动处理
if (destFolder) {
  const folder = await ensureFolder(destFolder);
  finalPath = `${folder}/${title}`;
}
finalPath = await uniquePath(finalPath);
await tp.file.move(finalPath);
-%>

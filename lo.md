# Life OS

> A self-contained dashboard for an Obsidian vault. It uses **Dataview** (`dataviewjs`) and gracefully shows empty states until the optional `Weekly/`, `Learning/`, `Finance/`, and `Events/` notes are created.

```dataviewjs
// Status bar — API requests are deliberately optional: static fallbacks keep the dashboard useful offline.
const ACCENT = "#7c5cff", ACCENT_DONE = "#33c481", ACCENT_LEARN = "#34b7eb", ACCENT_FOCUS = "#ff9f43", ACCENT_TODAY = "#ff5c8a";
const TODAY = window.moment().format("YYYY-MM-DD");
const root = dv.el("section", "", { cls: "lo lo-status" });
root.innerHTML = `<style>
.lo{--accent:${ACCENT};--done:${ACCENT_DONE};--learn:${ACCENT_LEARN};--focus:${ACCENT_FOCUS};--today:${ACCENT_TODAY};font-family:var(--font-interface);color:var(--text-normal)}
.lo-status{display:grid;grid-template-columns:1.3fr repeat(5,1fr);gap:10px;margin:0 0 18px}.lo-stat{border:1px solid var(--background-modifier-border);border-radius:12px;padding:11px 13px;background:var(--background-primary-alt);min-width:0}.lo-stat b{display:block;font-size:.72rem;text-transform:uppercase;letter-spacing:.06em;color:var(--text-muted)}.lo-stat span{display:block;margin-top:4px;font-weight:650;white-space:nowrap;overflow:hidden;text-overflow:ellipsis}.lo-count{border-color:color-mix(in srgb,var(--today),transparent 65%)}@media(max-width:767px){.lo-status{grid-template-columns:repeat(2,1fr)}.lo-status .lo-stat:first-child{grid-column:span 2}}
</style><div class="lo-stat"><b>Weather</b><span data-weather>☁️ Weather unavailable</span></div><div class="lo-stat"><b>Index</b><span data-index>—</span></div><div class="lo-stat"><b>SSE</b><span data-sse>—</span></div><div class="lo-stat"><b>DXY</b><span data-dxy>—</span></div><div class="lo-stat"><b>Gold</b><span data-gold>—</span></div><div class="lo-stat lo-count"><b>Next event</b><span data-count>None scheduled</span></div>`;
const set = (key, value) => root.querySelector(`[data-${key}]`).textContent = value;
const eventFiles = app.vault.getMarkdownFiles().filter(f => /(^|\/)(Weekly|Events)\//.test(f.path));
const events = [];
for (const file of eventFiles) { const cache = app.metadataCache.getFileCache(file); for (const item of cache?.listItems ?? []) { const text = item.task ?? item.text ?? ""; const date = text.match(/📅\s*(\d{4}-\d{2}-\d{2})/); if (date && date[1] >= TODAY) events.push({ date: date[1], text: text.replace(/📅\s*\d{4}-\d{2}-\d{2}/, "").replace(/^\[[ xX]\]\s*/, "") }); } }
events.sort((a,b) => a.date.localeCompare(b.date));
if (events[0]) { const days = window.moment(events[0].date).diff(window.moment(TODAY), "days"); set("count", `${days === 0 ? "Today" : `${days}d`} · ${events[0].text}`); }
// Free, CORS-friendly Open-Meteo request; failure retains the fallback above.
fetch("https://api.open-meteo.com/v1/forecast?latitude=0&longitude=0&current=temperature_2m,weather_code").then(r => r.ok ? r.json() : Promise.reject()).then(d => set("weather", `🌡️ ${Math.round(d.current.temperature_2m)}°C · ${d.current.weather_code === 0 ? "Clear" : "Current"}`)).catch(() => {});
// Quote APIs vary by deployment/CORS, so these intentionally fail soft rather than blocking the page.
fetch("https://query1.finance.yahoo.com/v7/finance/quote?symbols=%5EGSPC,000001.SS,DX-Y.NYB,GC%3DF").then(r => r.ok ? r.json() : Promise.reject()).then(d => { const q = d.quoteResponse.result; const fmt = s => s ? `${s.regularMarketPrice?.toLocaleString?.() ?? "—"}` : "—"; set("index", fmt(q.find(x => x.symbol === "^GSPC"))); set("sse", fmt(q.find(x => x.symbol === "000001.SS"))); set("dxy", fmt(q.find(x => x.symbol === "DX-Y.NYB"))); set("gold", fmt(q.find(x => x.symbol === "GC=F"))); }).catch(() => {});
```

## Daily Focus

```dataviewjs
const ACCENT_FOCUS = "#ff9f43";
const focusFile = "Daily Focus.md";
const host = dv.el("section", "", { cls: "lo lo-focus" });
host.innerHTML = `<style>.lo-focus{border-left:4px solid ${ACCENT_FOCUS};padding:14px 16px;margin-bottom:18px;background:var(--background-primary-alt);border-radius:0 12px 12px 0}.lo-focus label{font-size:.75rem;font-weight:700;color:var(--text-muted);text-transform:uppercase;letter-spacing:.08em}.lo-focus textarea{display:block;width:100%;margin-top:7px;background:transparent;border:0;resize:vertical;min-height:30px;font:600 1.06rem var(--font-interface);color:var(--text-normal);outline:none}.lo-focus small{color:var(--text-muted)}</style><label>Today’s focus</label><textarea placeholder="What one thing matters most today?"></textarea><small>Saved to ${focusFile} when you leave this field.</small>`;
const input = host.querySelector("textarea"); const file = app.vault.getAbstractFileByPath(focusFile);
if (file) input.value = (await app.vault.read(file)).match(/^focus:\s*(.*)$/m)?.[1] ?? "";
input.addEventListener("blur", async () => { const content = `---\nfocus: ${input.value.replace(/\n/g, " ")}\nupdated: ${window.moment().format("YYYY-MM-DD HH:mm")}\n---\n`; const existing = app.vault.getAbstractFileByPath(focusFile); existing ? await app.vault.modify(existing, content) : await app.vault.create(focusFile, content); });
```

## Weekly Checklist

```dataviewjs
const ACCENT_DONE = "#33c481";
const weekPath = `Weekly/${window.moment().format("GGGG-[W]WW")}.md`;
const weekHost = dv.el("section", "", { cls: "lo lo-weekly" });
weekHost.innerHTML = `<style>.lo-weekly{border:1px solid var(--background-modifier-border);padding:16px;border-radius:14px;margin-bottom:18px}.lo-week-head{display:flex;align-items:center;gap:14px}.lo-ring{width:48px;height:48px;flex:0 0 48px;transform:rotate(-90deg)}.lo-ring circle{fill:none;stroke-width:5}.lo-ring .base{stroke:var(--background-modifier-border)}.lo-ring .value{stroke:${ACCENT_DONE};stroke-linecap:round;transition:stroke-dashoffset .3s}.lo-weekly ul{list-style:none;padding:0;margin:14px 0 0}.lo-weekly li{padding:5px 0}.lo-weekly input{accent-color:${ACCENT_DONE};margin-right:8px}</style><div class="lo-week-head"><svg class="lo-ring" viewBox="0 0 42 42"><circle class="base" cx="21" cy="21" r="16"/><circle class="value" cx="21" cy="21" r="16"/></svg><div><b>This week</b><div class="muted" data-progress>Loading…</div></div></div><ul></ul>`;
const weeklyFile = app.vault.getAbstractFileByPath(weekPath); let weeklyText = weeklyFile ? await app.vault.read(weeklyFile) : "";
const tasks = [...weeklyText.matchAll(/^\s*- \[([ xX])\]\s+(.+)$/gm)].map(m => ({ done: m[1] !== " ", text: m[2] }));
const renderWeek = () => { const done = tasks.filter(t => t.done).length, total = tasks.length, pct = total ? done / total : 0, circle = weekHost.querySelector(".value"), c = 2 * Math.PI * 16; circle.style.strokeDasharray = c; circle.style.strokeDashoffset = c * (1 - pct); weekHost.querySelector("[data-progress]").textContent = total ? `${done}/${total} complete` : `No checklist yet — create ${weekPath}`; const list = weekHost.querySelector("ul"); list.replaceChildren(...tasks.map((t,i) => { const li = document.createElement("li"), box = document.createElement("input"); box.type = "checkbox"; box.checked = t.done; box.onchange = async () => { t.done = box.checked; weeklyText = weeklyText.replace(/^\s*- \[[ xX]\]\s+(.+)$/m, (line, text) => text === t.text ? `- [${t.done ? "x" : " "}] ${text}` : line); await app.vault.modify(weeklyFile, weeklyText); renderWeek(); }; li.append(box, document.createTextNode(t.text)); return li; })); }; renderWeek();
```

## Learning Plan

```dataviewjs
const ACCENT_LEARN = "#34b7eb";
const learnHost = dv.el("section", "", { cls: "lo lo-learn" });
learnHost.innerHTML = `<style>.lo-learn{display:grid;grid-template-columns:repeat(auto-fit,minmax(230px,1fr));gap:12px;margin-bottom:18px}.lo-course{border:1px solid var(--background-modifier-border);border-radius:12px;padding:14px}.lo-course h3{margin:0 0 8px;font-size:1rem}.lo-bar{height:7px;border-radius:99px;background:var(--background-modifier-border);overflow:hidden}.lo-bar i{display:block;height:100%;background:${ACCENT_LEARN};border-radius:inherit}.lo-course ul{list-style:none;margin:10px 0 0;padding:0}.lo-course li{display:flex;gap:7px;padding:3px 0}.lo-course input[type=text]{min-width:0;flex:1;border:0;border-bottom:1px dashed var(--background-modifier-border);background:transparent;color:var(--text-normal)}.lo-course input[type=checkbox]{accent-color:${ACCENT_LEARN}}</style>`;
const learningFiles = app.vault.getMarkdownFiles().filter(f => f.path.startsWith("Learning/"));
if (!learningFiles.length) learnHost.innerHTML += `<div class="lo-course"><h3>Learning Plan</h3><p class="muted">No notes found. Add task lists to <code>Learning/</code>; each file becomes a category.</p></div>`;
for (const file of learningFiles) { let text = await app.vault.read(file); const tasks = [...text.matchAll(/^\s*- \[([ xX])\]\s+(.+)$/gm)].map(m => ({ done:m[1]!==" ", text:m[2] })); const card = document.createElement("article"); card.className="lo-course"; const draw = () => { const done=tasks.filter(t=>t.done).length, pct=tasks.length ? 100*done/tasks.length : 0; card.innerHTML=`<h3>${file.basename}</h3><div class="lo-bar"><i style="width:${pct}%"></i></div><small>${done}/${tasks.length} complete</small><ul></ul>`; const ul=card.querySelector("ul"); tasks.forEach(t=>{const li=document.createElement("li"), box=document.createElement("input"), edit=document.createElement("input"); box.type="checkbox";box.checked=t.done;edit.type="text";edit.value=t.text; const save=async()=>{text=text.replace(/^\s*- \[[ xX]\]\s+(.+)$/m,(line,name)=>name===t.text?`- [${t.done?"x":" "}] ${name}`:line);await app.vault.modify(file,text);draw();}; box.onchange=async()=>{t.done=box.checked;await save()}; edit.onchange=async()=>{const old=t.text;t.text=edit.value;text=text.replace(new RegExp(`^- \\[[ xX]\\] ${old.replace(/[.*+?^${}()|[\]\\]/g,"\\$&")}$`,"m"),`- [${t.done?"x":" "}] ${t.text}`);await app.vault.modify(file,text);draw()};li.append(box,edit);ul.append(li)})}; draw(); learnHost.append(card); }
```

## Events Calendar

```dataviewjs
const ACCENT_TODAY = "#ff5c8a";
const cal = dv.el("section", "", { cls: "lo lo-calendar" }); const month = window.moment();
cal.innerHTML = `<style>.lo-calendar{margin-bottom:18px}.lo-cal-grid{display:grid;grid-template-columns:repeat(7,1fr);gap:5px}.lo-cal-label{text-align:center;color:var(--text-muted);font-size:.75rem}.lo-day{min-height:84px;border:1px solid var(--background-modifier-border);border-radius:8px;padding:6px;font-size:.8rem}.lo-day.today{border-color:${ACCENT_TODAY};box-shadow:inset 0 0 0 1px ${ACCENT_TODAY}}.lo-event{display:block;margin-top:3px;padding:2px 4px;border-radius:4px;background:color-mix(in srgb,${ACCENT_TODAY},transparent 84%);overflow:hidden;text-overflow:ellipsis;white-space:nowrap}@media(max-width:767px){.lo-day{min-height:54px}.lo-event{font-size:.67rem}.lo-cal-grid{gap:2px}}</style><h3>${month.format("MMMM YYYY")}</h3><div class="lo-cal-grid"></div>`;
const calEvents = {}; for (const f of app.vault.getMarkdownFiles().filter(f=>/(^|\/)(Weekly|Events)\//.test(f.path))) { const txt=await app.vault.read(f); for (const m of txt.matchAll(/^- \[[ xX]\]\s*(.*?)\s*📅\s*(\d{4}-\d{2}-\d{2})/gm)) (calEvents[m[2]]??=[]).push(m[1]); }
const grid=cal.querySelector(".lo-cal-grid"); ["Sun","Mon","Tue","Wed","Thu","Fri","Sat"].forEach(x=>grid.insertAdjacentHTML("beforeend",`<div class="lo-cal-label">${x}</div>`)); for(let i=0;i<month.clone().startOf("month").day();i++) grid.append(document.createElement("div")); for(let d=1;d<=month.daysInMonth();d++){const date=month.clone().date(d).format("YYYY-MM-DD"), cell=document.createElement("div");cell.className=`lo-day ${date===window.moment().format("YYYY-MM-DD")?"today":""}`;cell.innerHTML=`<b>${d}</b>${(calEvents[date]??[]).map(e=>`<span class="lo-event" title="${e}">${e}</span>`).join("")}`;grid.append(cell)}
```

## Finance Overview

```dataviewjs
const ACCENT = "#7c5cff";
const financeHost = dv.el("section", "", { cls: "lo lo-finance" }); const year = window.moment().format("YYYY"), monthFile=`Finance/${window.moment().format("YYYY-MM")}.md`;
financeHost.innerHTML=`<style>.lo-finance{border:1px solid var(--background-modifier-border);border-radius:14px;padding:16px}.lo-finance-top{display:flex;justify-content:space-between;gap:12px;align-items:center}.lo-finance a{color:${ACCENT};font-weight:650}.lo-goal{height:10px;margin:12px 0;border-radius:99px;overflow:hidden;background:var(--background-modifier-border)}.lo-goal i{display:block;height:100%;background:${ACCENT};border-radius:inherit}.lo-finance dl{display:grid;grid-template-columns:repeat(3,1fr);gap:10px;margin:0}.lo-finance dt{font-size:.75rem;color:var(--text-muted)}.lo-finance dd{margin:3px 0 0;font-size:1.15rem;font-weight:700}@media(max-width:767px){.lo-finance dl{grid-template-columns:1fr}}</style><div class="lo-finance-top"><div><b>${year} Finance</b><div class="muted">Yearly totals from Finance/ notes</div></div><a href="${monthFile}">Open this month ↗</a></div><div class="lo-goal"><i></i></div><dl><div><dt>Income</dt><dd data-income>0</dd></div><div><dt>Expenses</dt><dd data-expense>0</dd></div><div><dt>Net</dt><dd data-net>0</dd></div></dl><small data-goal>Set <code>goal:</code> in a Finance note to show a goal.</small>`;
let income=0,expense=0,goal=0; for(const f of app.vault.getMarkdownFiles().filter(f=>f.path.startsWith(`Finance/${year}`))){const txt=await app.vault.read(f); income+=Number(txt.match(/^income:\s*([\d.]+)/m)?.[1]??0);expense+=Number(txt.match(/^expense:\s*([\d.]+)/m)?.[1]??0);goal=Math.max(goal,Number(txt.match(/^goal:\s*([\d.]+)/m)?.[1]??0));} const money=n=>new Intl.NumberFormat(undefined,{style:"currency",currency:"USD",maximumFractionDigits:0}).format(n); financeHost.querySelector("[data-income]").textContent=money(income);financeHost.querySelector("[data-expense]").textContent=money(expense);financeHost.querySelector("[data-net]").textContent=money(income-expense);const pct=goal?Math.min(100,income/goal*100):0;financeHost.querySelector(".lo-goal i").style.width=`${pct}%`;financeHost.querySelector("[data-goal]").textContent=goal?`${money(income)} of ${money(goal)} income goal (${Math.round(pct)}%)`:`No Finance/${year} notes yet — totals are ready when you are.`;
```

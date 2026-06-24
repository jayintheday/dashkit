---
name: dashkit
description: "Build a live, single-file dashboard styled like the shadcn/ui dashboard-01 block, delivered as a self-contained Claude Artifact wired to the user's own remote MCP connectors so the numbers are real, not fake. Supports light and dark themes (dark by default). Use when someone asks for a dashboard, admin panel, metrics view, or status page over their connected services (Vercel, Supabase, Stripe, Linear, GitHub, Notion, Postgres, Google Sheets, …)."
---

# Dashkit — live shadcn/ui dashboard artifacts

You build a **single self-contained `.html` file** — inline CSS + inline JS, Chart.js from a
CDN, no build step, no framework — that looks like the shadcn/ui **dashboard-01** block
(inset sidebar, a row of stat cards, 1–2 charts, a tabbed paginated table) and shows **live
data pulled from the user's own connected services** at load time.

The difference from a static shadcn demo is the whole point: **the numbers are real.** Don't
build a pretty template with placeholder data — build a dashboard for *this* user's actual
data.

> **This file is self-sufficient.** The complete, styled skeleton is embedded below under
> **"Build kit"** — copy that whole block and replace the placeholders. **Do not hand-roll your
> own CSS from the prose; that's how you get generic AI-slop output instead of the shadcn look.**
> You cannot use the real shadcn React components in an Artifact (no build step), so this
> reproduces shadcn's look — its Neutral tokens, `new-york` style, Geist font, dashboard-01
> layout — in hand-written CSS, with Chart.js standing in for shadcn/Recharts charts.

> Optional depth, if present in the folder: `dashboard-template.html` is the same skeleton as a
> standalone file, and `DASHBOARD-PLAYBOOK.md` is extended reference (data-shape tables, deeper
> notes). Neither is required — if all you have is this SKILL.md, everything you need is here.

## The one thing that breaks everything if you miss it

The dashboard runs as a **Claude Artifact: a sandboxed iframe**. It does **not** use
`fetch()`. Data reaches it only through a bridge the runtime injects:

```js
const r = await window.cowork.callMcpTool(name, args);
```

That bridge can only reach **remote MCP connectors** (the URL/OAuth ones in Claude →
Settings → Connectors). It **cannot** reach a **local/stdio MCP server** — the sandbox can't
talk to the user's machine. This is the #1 setup-loop trap: the agent wires the dashboard to
a local Postgres/MCP, the artifact loads, every call silently fails, and the user just sees a
red "Couldn't load live data" box with no clue why.

**So check connectivity before building.** If the user's data lives only in a local MCP,
**stop and tell them to add a remote connector first** — don't build a dashboard that can't
reach its data. (If the bridge fails even with a remote connector wired, confirm the runtime's
bridge global is still `window.cowork.callMcpTool` — APIs drift.)

## Procedure — five phases, in order. Do not skip to Build.

The first three phases are what prevent the "dashboard that can't reach its data" loop.

### Phase 1 — Discover the connections
Look at the MCP tools **you (the agent) can call in this conversation** — those *are* the
remote connectors, and their `mcp__<server-uuid>__<operation>` names are exactly what the
artifact will call. Copy them verbatim; the server UUIDs are per-user/session, never guessed.
If the only data source is a local MCP, stop (see above).

### Phase 2 — Review what data they actually have
For each connected source, find what it exposes: call its `list_*` / read operations, or ask
the user for the IDs you're missing (a Vercel `projectId`, a Supabase `project_ref`) — never
invent them. Make **one cheap probe call per source** to confirm it returns data. Note the
real metrics available: counts, revenue, statuses, time series, rows. A source that errors on
probe is one you design *around*, not into.

### Phase 3 — Propose a smart layout, then confirm
Don't build a generic dashboard — design one for *this* project from the data you found:
- the handful of most decision-useful metrics → top **stat cards**
- 1–2 **charts** that fit the data you actually have (time series → area/line; categorical →
  bars; skip a chart if nothing suits it)
- what belongs in the **data table** tabs

Briefly propose this (which metrics, which charts, which tabs) and adjust to the user's
priorities **before** building.

### Phase 4 — Build the single file
**Start from the Build kit below — copy the whole skeleton, then make these edits.** It already
ships the shadcn tokens, the light/dark theme + toggle, the `call()` helper, and theme-aware
charts. Your job is to swap placeholders for real data, not to restyle.

- **Wire the data.** Replace the `SOURCE_*` / `*_ID` consts (top of the script) with the real
  tool names + IDs from Phase 1. In `load()`, replace `const data = SAMPLE` with values derived
  from `await call(TOOL, args)`. Run independent calls with `Promise.all`. Wrap each **optional**
  source in its own `try/catch` so one dead integration degrades gracefully (e.g. "Pre-revenue")
  instead of blanking the page. Convert money from cents. **Delete the whole `SAMPLE` block.**
- **Make it this project's dashboard.** Rename the stat cards, chart titles, and table columns to
  the metrics that matter. Delete any card/chart/tab you have no real data for.
- **Theme:** leave the token blocks alone — shadcn **Neutral**, `new-york`, radius 10px, light on
  `:root` + dark under `.dark`, dark by default. **Every color comes from a CSS variable; never
  hardcode one** (that's what breaks theming). Charts read tokens at draw time via the
  `cssVar()`/`hexToRgba()` helpers, so they follow the theme automatically — keep that pattern.
- **Keep the light/dark toggle — it's a required feature, not decoration.** The kit puts a working
  sun/moon button (`#themeToggle`) in the header that flips the `.dark` class, persists the choice
  to `localStorage`, and redraws the charts. **If you restyle or rebuild the header, the toggle must
  stay** — it's easy to drop it while adding a title/date/refresh and end up with dark-only and no
  switcher. The same goes for the other real controls the kit ships (sidebar nav scroll-to, tab
  switching, pagination, the chart range toggle, the row-select count): keep them working. The
  "no fake elements" rule below deletes *dead* chrome, never live controls like these.
- **Font (best-effort, don't block on it):** the kit loads **Geist** from jsdelivr (same CDN as
  Chart.js). The Artifact CSP may still block fonts; if so the system fallback applies and that's
  fine — the tokens + layout carry the look. Don't treat a missing webfont as a failure, and don't
  swap in a different font.
- **No fake elements** — the biggest AI-slop tell. If it looks clickable, it works, or it's gone.
  Wire real external links where you have the IDs (Vercel team, Supabase project, Stripe dashboard,
  commit → deployment URL); **delete** decorative chrome (Quick Create, Inbox, Settings, Search,
  "Customize Columns", "Add Section", drag-handles, dead ⋯ menus, `href="#"` logos). The kit's
  Resources block is commented out for this reason — uncomment and wire it only with real links.
- **Hygiene:** the kit already HTML-escapes live values (`esc()`); keep using it, and add
  `rel="noopener"` to every `target="_blank"`.

### Phase 5 — Verify
- **Inside the Claude artifact runtime** (normal case): `window.cowork` is live, so the
  **live preview *is* the test** — load it and confirm real data populates, both themes look
  right (toggle and watch the charts re-read their colors), and nothing interactive is dead. If
  data doesn't load, re-check Phase 1 (remote vs local connector) first.
- **In a coding agent on disk** (`window.cowork` doesn't exist): the kit renders from its own
  `SAMPLE` block so you can screenshot it directly; for a wired build, mock `window.cowork`
  (see `DASHBOARD-PLAYBOOK.md` §6 if present). Delete any throwaway preview file after.

## Pre-ship checklist
- [ ] Data source is a **remote** MCP connector (not local/stdio).
- [ ] Tool-name consts came from the agent's own connected tools; IDs confirmed via a probe.
- [ ] Layout chosen from data that actually exists and run past the user — not a template.
- [ ] Single file; only external deps are the Chart.js + Geist CDN tags.
- [ ] All colors from CSS tokens; light on `:root`, dark under `.dark`, each with `color-scheme`.
- [ ] A **visible light/dark toggle is present in the header** and works (survived any header
  restyle); **charts re-read their colors on switch** (no hardcoded JS colors).
- [ ] Charts, tooltips, gridlines, axis ticks all legible in **both** light and dark.
- [ ] Every interactive-looking element does something real, or is gone.
- [ ] External links correct, new tab, `rel="noopener"`.
- [ ] Live values HTML-escaped; amounts converted from cents.
- [ ] Optional integrations fail gracefully (no whole-page blank on one dead source).
- [ ] Empty-state `colspan` matches the final column count (the kit computes it from the headers).
- [ ] All `SAMPLE`/placeholder data and `SOURCE_*`/`REPLACE_ME` consts removed; real values in.
- [ ] Real IDs/email reviewed before any sharing.

## Build kit — copy this whole file, then do Phase 4

This is a complete, self-contained scaffold: light/dark shadcn dashboard-01 with a theme toggle,
the `call()` MCP normalizer, and theme-aware Chart.js. It renders out of the box on `SAMPLE` data
so you can see it; replace the placeholders to make it live. **Copy it verbatim — don't
reconstruct the CSS by hand.**

```html
<!DOCTYPE html>
<!--
  dashkit — clean shadcn/ui dashboard-01 scaffold (light + dark).
  This is a STARTING POINT, not a finished dashboard. See SKILL.md for the procedure.

  To turn it into a live dashboard:
    1. Replace the SOURCE_* / *_ID consts below with the real MCP tool names + IDs you
       discovered (SKILL.md Phase 1). They look like  mcp__<server-uuid>__<operation>.
    2. Replace the SAMPLE_* assignments in load() with real `await call(TOOL, args)` results
       (the call() helper is already here). SAMPLE_* is preview-only filler — delete it.
    3. Rename the stat cards / chart titles / table columns to the metrics that matter for
       THIS project. Delete cards/charts/tabs you have no real data for — never ship filler.
  Theme: shadcn Neutral, new-york, radius 10px. Dark by default; toggle in the header.
-->
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>Dashboard</title>
<!-- set the theme before first paint so there's no flash -->
<script>(function(){try{var t=localStorage.getItem("dashkit_theme")||"dark";if(t!=="light")document.documentElement.classList.add("dark");}catch(e){document.documentElement.classList.add("dark");}})();</script>
<!-- Geist via jsdelivr (the same CDN that serves Chart.js below, so it clears the same allowlist).
     If your artifact sandbox blocks these too, the system fallback in body{} takes over — that's
     acceptable: the tokens + layout carry the shadcn look, the font is the last 10%.
     Google Fonts alt (family "Geist"): https://fonts.googleapis.com/css2?family=Geist:wght@300..700 -->
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@fontsource/geist-sans@5/400.css">
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@fontsource/geist-sans@5/500.css">
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@fontsource/geist-sans@5/600.css">
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@fontsource/geist-sans@5/700.css">
<script src="https://cdn.jsdelivr.net/npm/chart.js@4.5.0/dist/chart.umd.js" integrity="sha384-iU8HYtnGQ8Cy4zl7gbNMOhsDTTKX02BTXptVP/vqAWIaTfM7isw76iyZCsjL2eVi" crossorigin="anonymous"></script>
<style>
/* ===== shadcn/ui new-york v4 tokens (Neutral) — light in :root, dark in .dark ===== */
:root{
  color-scheme:light;
  --background:#ffffff; --foreground:#0a0a0a;
  --card:#ffffff; --card-foreground:#0a0a0a;
  --muted:#f5f5f5; --muted-foreground:#737373;
  --border:#e5e5e5; --input:#e5e5e5; --ring:#0a0a0a;
  --primary:#0a0a0a; --primary-foreground:#fafafa;
  --secondary:#f5f5f5; --accent:#f5f5f5; --accent-foreground:#0a0a0a;
  --destructive:#ef4444;
  --sidebar:#fafafa; --sidebar-foreground:#0a0a0a; --sidebar-accent:#f0f0f0;
  --sidebar-border:#e5e5e5; --sidebar-primary:#0a0a0a;
  --green:#16a34a;
  /* shadcn chart palette (light) */
  --chart-1:#e76e50; --chart-2:#2a9d90; --chart-3:#274754; --chart-4:#e8c468; --chart-5:#f4a462;
  /* theme-aware accents that CSS uses directly */
  --scard-top:rgba(10,10,10,.04);     /* card top-gradient = primary/5 */
  --row-hover:rgba(10,10,10,.03);
  --radius:10px; --radius-lg:10px; --radius-md:8px; --radius-sm:6px;
  --shadow-sm:0 1px 3px 0 rgba(0,0,0,.1),0 1px 2px -1px rgba(0,0,0,.1);
  --shadow-xs:0 1px 2px 0 rgba(0,0,0,.05);
  --header-height:48px;
}
.dark{
  color-scheme:dark;
  --background:#0a0a0a; --foreground:#fafafa;
  --card:#1c1c1c; --card-foreground:#fafafa;           /* cards LIGHTER than bg = elevated */
  --muted:#272727; --muted-foreground:#a1a1a1;
  --border:#2a2a2a; --input:#2f2f2f; --ring:#8a8a8a;
  --primary:#fafafa; --primary-foreground:#0a0a0a;     /* primary INVERTS in dark */
  --secondary:#272727; --accent:#272727; --accent-foreground:#fafafa;
  --destructive:#ff5b5b;
  --sidebar:#1c1c1c; --sidebar-foreground:#fafafa; --sidebar-accent:#2a2a2a;
  --sidebar-border:#2a2a2a; --sidebar-primary:#fafafa;
  --green:#22c55e;
  /* shadcn chart palette (dark) */
  --chart-1:#2662d9; --chart-2:#2eb88a; --chart-3:#e88c30; --chart-4:#af57db; --chart-5:#e23670;
  --scard-top:rgba(255,255,255,.05);
  --row-hover:rgba(255,255,255,.04);
  --shadow-sm:0 1px 3px 0 rgba(0,0,0,.4),0 1px 2px -1px rgba(0,0,0,.4);
  --shadow-xs:0 1px 2px 0 rgba(0,0,0,.3);
}
*{box-sizing:border-box}
html,body{margin:0;padding:0}
body{background:var(--sidebar);color:var(--foreground);
  font-family:"Geist Sans","Geist",ui-sans-serif,system-ui,-apple-system,BlinkMacSystemFont,"Segoe UI",Roboto,Helvetica,Arial,sans-serif;
  font-size:14px;line-height:1.45;-webkit-font-smoothing:antialiased}
button{font:inherit;cursor:pointer}
svg{display:block}
a{color:inherit;text-decoration:none}

.wrapper{display:flex;min-height:100vh;background:var(--sidebar)}

/* ===== Sidebar (inset) ===== */
.sidebar{width:256px;flex:none;background:var(--sidebar);color:var(--sidebar-foreground);
  display:flex;flex-direction:column;gap:8px;padding:8px;position:sticky;top:0;height:100vh}
.smenu-btn{display:flex;align-items:center;gap:8px;height:32px;padding:0 8px;border-radius:var(--radius-md);
  font-size:13.5px;color:var(--sidebar-foreground);border:1px solid transparent}
.smenu-btn:hover{background:var(--sidebar-accent)}
.smenu-btn.active{background:var(--sidebar-accent);font-weight:500}
.smenu-btn svg{width:16px;height:16px;color:var(--muted-foreground);flex:none}
.smenu-btn.active svg{color:var(--foreground)}
.sb-head .smenu-btn{height:auto;padding:6px 8px}
.sb-head .mark{width:24px;height:24px;border-radius:6px;background:var(--primary);color:var(--primary-foreground);
  display:flex;align-items:center;justify-content:center;font-weight:700;font-size:13px;flex:none}
.sb-head .ttl{font-size:15px;font-weight:600}
.sb-content{display:flex;flex-direction:column;gap:8px;flex:1;overflow-y:auto}
.sb-group{display:flex;flex-direction:column;gap:2px}
.sb-label{font-size:11.5px;color:var(--muted-foreground);font-weight:500;height:28px;display:flex;align-items:center;padding:0 8px}
.sb-foot{margin-top:auto;padding-top:2px}
.user-btn{display:flex;align-items:center;gap:8px;height:48px;padding:0 8px;border-radius:var(--radius-md)}
.user-btn .av{width:32px;height:32px;border-radius:var(--radius-md);background:var(--muted);
  display:flex;align-items:center;justify-content:center;font-size:12px;font-weight:600;flex:none}
.user-btn .ut{flex:1;min-width:0;line-height:1.25}
.user-btn .ut b{font-size:13px;font-weight:500;display:block;overflow:hidden;text-overflow:ellipsis;white-space:nowrap}
.user-btn .ut span{font-size:11.5px;color:var(--muted-foreground);display:block;overflow:hidden;text-overflow:ellipsis;white-space:nowrap}

/* ===== Inset main ===== */
.inset{flex:1;min-width:0;background:var(--background);margin:8px;margin-left:0;border-radius:14px;
  box-shadow:var(--shadow-sm);display:flex;flex-direction:column;overflow:hidden;border:1px solid var(--border)}
.site-header{height:var(--header-height);flex:none;display:flex;align-items:center;gap:8px;
  padding:0 16px;border-bottom:1px solid var(--border)}
.ghost-icon{width:28px;height:28px;border-radius:var(--radius-md);display:flex;align-items:center;justify-content:center;
  background:transparent;color:var(--foreground);border:0}
.ghost-icon:hover{background:var(--accent)}.ghost-icon svg{width:18px;height:18px}
.vsep{width:1px;height:16px;background:var(--border);margin:0 8px}
.site-header h1{font-size:16px;font-weight:500;margin:0}
.site-header .right{margin-left:auto;display:flex;align-items:center;gap:8px}

.scroll{flex:1;overflow-y:auto}
.main-col{display:flex;flex-direction:column;gap:24px;padding:24px 0}

/* ===== Badge ===== */
.badge{display:inline-flex;align-items:center;justify-content:center;gap:4px;width:fit-content;
  font-size:12px;font-weight:500;line-height:16px;border:1px solid transparent;border-radius:999px;
  padding:2px 8px;white-space:nowrap}
.badge svg{width:12px;height:12px}
.badge.outline{border-color:var(--border);color:var(--foreground)}
.badge.muted{border-color:var(--border);color:var(--muted-foreground);padding:2px 6px}

/* ===== Section cards ===== */
.section-cards{display:grid;grid-template-columns:repeat(4,1fr);gap:16px;padding:0 24px}
@media(max-width:1200px){.section-cards{grid-template-columns:repeat(2,1fr)}}
@media(max-width:600px){.section-cards{grid-template-columns:1fr}}
.card{background:var(--card);border:1px solid var(--border);border-radius:14px;box-shadow:var(--shadow-xs)}
.scard{display:flex;flex-direction:column;gap:24px;padding:24px 0;
  background-image:linear-gradient(to top, var(--scard-top), var(--card))}
.scard .hd{display:grid;grid-template-columns:1fr auto;align-items:start;row-gap:8px;padding:0 24px}
.scard .desc{grid-column:1;font-size:14px;color:var(--muted-foreground)}
.scard .title{grid-column:1;font-size:30px;font-weight:600;line-height:1;font-variant-numeric:tabular-nums}
.scard .act{grid-column:2;grid-row:1 / span 2;align-self:start;justify-self:end}
.scard .ft{display:flex;flex-direction:column;align-items:flex-start;gap:6px;padding:0 24px;font-size:14px}
.scard .ft .l1{display:flex;align-items:center;gap:8px;font-weight:500}
.scard .ft .l1 svg{width:16px;height:16px}
.scard .ft .l2{color:var(--muted-foreground)}

/* ===== Chart card ===== */
.block{padding:0 24px}
.chart-card{display:flex;flex-direction:column;gap:24px;padding:24px 0}
.chart-head{display:grid;grid-template-columns:1fr auto;align-items:start;gap:8px;padding:0 24px}
.chart-head .ct{font-size:15px;font-weight:600;line-height:1}
.chart-head .cs{font-size:14px;color:var(--muted-foreground);margin-top:6px}
.chart-content{padding:0 24px}
.chart-box{position:relative;height:250px;width:100%}
.chart-grid{display:grid;grid-template-columns:1.6fr 1fr;gap:24px}
@media(max-width:1000px){.chart-grid{grid-template-columns:1fr}}
.tgroup{display:inline-flex;border:1px solid var(--border);border-radius:var(--radius-md);overflow:hidden;box-shadow:var(--shadow-xs)}
.tgroup button{height:32px;padding:0 16px;background:var(--background);color:var(--foreground);font-size:13px;font-weight:500;
  border:0;border-right:1px solid var(--border)}
.tgroup button:last-child{border-right:0}
.tgroup button:hover{background:var(--accent)}
.tgroup button.active{background:var(--accent)}
@media(max-width:720px){.tgroup button{padding:0 10px}}

/* ===== Tabs (data table) ===== */
.dt{display:flex;flex-direction:column;gap:24px}
.dt-top{display:flex;align-items:center;justify-content:space-between;gap:12px;padding:0 24px;flex-wrap:wrap}
.tabslist{display:inline-flex;align-items:center;background:var(--muted);border-radius:var(--radius-lg);padding:3px;height:36px}
.tabslist button{display:inline-flex;align-items:center;gap:6px;height:100%;padding:0 10px;border:1px solid transparent;border-radius:var(--radius-md);
  background:transparent;color:var(--muted-foreground);font-size:13px;font-weight:500}
.tabslist button:hover{color:var(--foreground)}
.tabslist button.active{background:var(--background);color:var(--foreground);box-shadow:var(--shadow-sm)}
.tabslist .b{font-size:11px;background:rgba(115,115,115,.3);border-radius:999px;height:20px;min-width:20px;
  display:inline-flex;align-items:center;justify-content:center;padding:0 4px}

.tbl-wrap{margin:0 24px;border:1px solid var(--border);border-radius:var(--radius-lg);overflow:hidden}
table{width:100%;border-collapse:collapse;font-size:13.5px}
thead{background:var(--muted)}
th{height:40px;padding:0 8px;text-align:left;font-weight:500;color:var(--foreground);white-space:nowrap;border-bottom:1px solid var(--border)}
td{padding:8px;vertical-align:middle;border-bottom:1px solid var(--border);white-space:nowrap}
tbody tr:last-child td{border-bottom:0}
tbody tr:hover td{background:var(--row-hover)}
th.mid,td.mid{width:32px;text-align:center;padding:0}
input[type=checkbox]{width:16px;height:16px;accent-color:var(--primary);margin:0;vertical-align:middle;cursor:pointer}
.mut{color:var(--muted-foreground)}
.statusb svg{width:13px;height:13px}
.statusb .ok{color:var(--green)}.statusb .ld{color:var(--muted-foreground)}.statusb .er{color:var(--destructive)}

.dt-foot{display:flex;align-items:center;justify-content:space-between;gap:16px;padding:0 24px 4px;flex-wrap:wrap}
.dt-foot .sel{font-size:13.5px;color:var(--muted-foreground);flex:1}
.dt-foot .grp{display:flex;align-items:center;gap:32px}
.dt-foot .rpp{display:flex;align-items:center;gap:8px;font-size:13.5px;font-weight:500}
select.sel-sm{height:32px;border:1px solid var(--border);border-radius:var(--radius-md);background:var(--background);
  color:var(--foreground);box-shadow:var(--shadow-xs);font:inherit;font-size:13px;padding:0 8px}
.pageinfo{font-size:13.5px;font-weight:500}
.pager{display:flex;gap:8px}
.pgbtn{width:32px;height:32px;border:1px solid var(--border);border-radius:var(--radius-md);background:var(--background);
  box-shadow:var(--shadow-xs);display:flex;align-items:center;justify-content:center;color:var(--foreground)}
.pgbtn:hover{background:var(--accent)}.pgbtn:disabled{opacity:.5;cursor:not-allowed}.pgbtn svg{width:15px;height:15px}

.footnote{font-size:11.5px;color:var(--muted-foreground);line-height:1.6;padding:0 24px 8px}
.loading,.errbox{padding:48px;text-align:center;color:var(--muted-foreground)}
.errbox{color:var(--destructive)}

@media(max-width:860px){
  .wrapper{flex-direction:column}
  .sidebar{width:100%;height:auto;position:static;flex-direction:row;align-items:center;overflow-x:auto;border-bottom:1px solid var(--sidebar-border)}
  .sb-content,.sb-foot{display:none}.inset{margin:0;border-radius:0;border:0}
}
</style>
</head>
<body>
<div class="wrapper">
  <!-- ===== Sidebar ===== -->
  <aside class="sidebar">
    <div class="sb-head">
      <a class="smenu-btn"><span class="mark">D</span><span class="ttl">Dashboard</span></a>
    </div>
    <div class="sb-content">
      <div class="sb-group">
        <a class="smenu-btn active" data-go="top"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><rect x="3" y="3" width="7" height="7" rx="1"/><rect x="14" y="3" width="7" height="7" rx="1"/><rect x="14" y="14" width="7" height="7" rx="1"/><rect x="3" y="14" width="7" height="7" rx="1"/></svg>Overview</a>
        <a class="smenu-btn" data-go="chart"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M3 3v18h18"/><path d="M7 15l4-4 3 3 5-6"/></svg>Analytics</a>
        <a class="smenu-btn" data-go="table"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><rect x="2" y="5" width="20" height="14" rx="2"/><path d="M2 10h20"/></svg>Records</a>
      </div>
      <!--
        "Resources" links are external <a target="_blank"> to the real dashboards behind your
        data sources. Build the URLs from IDs you already have, and DELETE this whole group if
        you have no real links — do not ship href="#" placeholders.
        <div class="sb-group">
          <div class="sb-label">Resources</div>
          <a class="smenu-btn" href="https://example.com/console" target="_blank" rel="noopener">…</a>
        </div>
      -->
    </div>
    <div class="sb-foot">
      <div class="user-btn">
        <span class="av">U</span>
        <span class="ut"><b>Your Name</b><span>you@example.com</span></span>
      </div>
    </div>
  </aside>

  <!-- ===== Inset ===== -->
  <div class="inset">
    <header class="site-header">
      <button class="ghost-icon" id="sbToggle" title="Toggle Sidebar"><svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><rect x="3" y="3" width="18" height="18" rx="2"/><path d="M9 3v18"/></svg></button>
      <div class="vsep"></div>
      <h1>Overview</h1>
      <div class="right">
        <span class="badge outline" id="hdrStatus">Loading…</span>
        <button class="ghost-icon" id="themeToggle" title="Toggle theme" aria-label="Toggle theme"></button>
      </div>
    </header>
    <div class="scroll" id="top">
      <div class="main-col" id="app">
        <div class="loading">Loading…</div>
      </div>
    </div>
  </div>
</div>

<script>
/* ═══════════════ 1. WIRE YOUR DATA SOURCES (SKILL.md Phase 1) ═══════════════
   These are placeholders. Replace with the real MCP tool names + IDs from your own
   connected remote connectors. The artifact reaches REMOTE connectors only. */
const SOURCE_A = "mcp__SERVER_UUID__list_items";   // ← replace; e.g. mcp__<uuid>__list_deployments
const SOURCE_B = "mcp__SERVER_UUID__get_metrics";  // ← replace; delete if you only have one source
const PROJECT_ID = "REPLACE_ME";                   // ← replace, or delete if the tool needs no IDs

/* ═══════════════ 2. PREVIEW-ONLY SAMPLE DATA — DELETE WHEN WIRED ═══════════════
   Lets the scaffold render before it's connected. In a real build, remove this and
   populate from `await call(...)` in load(). Never ship a dashboard on sample data. */
const SAMPLE = {
  stats:{ revenue: 48250.00, users: 1284, records: 342, success: 98 },
  // time series for the area chart (last 90 days, two stacked series)
  series:(function(){const a=[];const now=Date.now();for(let i=89;i>=0;i--){const t=now-i*864e5;
    a.push({t, primary: Math.round(6+8*Math.random()+ (i<30?6:0)), secondary: Math.round(2+4*Math.random())});}return a;})(),
  breakdown:[ {label:"Organic",value:540},{label:"Referral",value:312},{label:"Direct",value:268},{label:"Email",value:164} ],
  records:[
    {name:"Initial import",status:"READY",owner:"Ada",when:Date.now()-2*864e5},
    {name:"Schema migration",status:"READY",owner:"Lin",when:Date.now()-5*864e5},
    {name:"Backfill job",status:"RUNNING",owner:"Sam",when:Date.now()-6*864e5},
    {name:"Failed export",status:"ERROR",owner:"Ada",when:Date.now()-9*864e5}
  ],
  items:[
    {label:"Workspaces",count:128,note:"public"},
    {label:"Members",count:1284,note:"public"},
    {label:"Projects",count:342,note:"public"},
    {label:"Integrations",count:17,note:"public"}
  ]
};

/* ═══════════════ helpers (keep all of these) ═══════════════ */
const cssVar = n => getComputedStyle(document.documentElement).getPropertyValue(n).trim();
function hexToRgba(hex,a){hex=(hex||"").replace("#","");if(hex.length===3)hex=hex.split("").map(c=>c+c).join("");
  const n=parseInt(hex||"000000",16);return "rgba("+((n>>16)&255)+","+((n>>8)&255)+","+(n&255)+","+a+")";}
const esc = s => (s==null?"":String(s)).replace(/[&<>"]/g,c=>({"&":"&amp;","<":"&lt;",">":"&gt;","\"":"&quot;"}[c]));
const money = v => "$"+v.toLocaleString(undefined,{minimumFractionDigits:2,maximumFractionDigits:2});
const num = v => v==null?"—":Number(v).toLocaleString();
const fmtDate = ms => new Date(ms).toLocaleDateString(undefined,{day:"numeric",month:"short",year:"numeric"});
const dayKey = ms => {const d=new Date(ms);d.setHours(0,0,0,0);return d.getTime();};
const weekKey = ms => {const d=new Date(ms);const w=(d.getDay()+6)%7;d.setHours(0,0,0,0);d.setDate(d.getDate()-w);return d.getTime();};
if(window.Chart)Chart.defaults.font.family='"Geist Sans","Geist",ui-sans-serif,system-ui,-apple-system,BlinkMacSystemFont,"Segoe UI",Roboto,Helvetica,Arial,sans-serif';

const SVG={
  up:'<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M16 7h6v6"/><path d="m22 7-8.5 8.5-5-5L2 17"/></svg>',
  down:'<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M16 17h6v-6"/><path d="m22 17-8.5-8.5-5 5L2 7"/></svg>',
  check:'<svg viewBox="0 0 24 24" fill="currentColor"><path d="M12 2a10 10 0 1 0 0 20 10 10 0 0 0 0-20Zm-1.2 14.2-3.5-3.5 1.4-1.4 2.1 2.1 4.5-4.5 1.4 1.4-5.9 5.9Z"/></svg>',
  loader:'<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M12 2v4"/><path d="M12 18v4"/><path d="m4.9 4.9 2.9 2.9"/><path d="m16.2 16.2 2.9 2.9"/><path d="M2 12h4"/><path d="M18 12h4"/><path d="m4.9 19.1 2.9-2.9"/><path d="m16.2 7.8 2.9-2.9"/></svg>',
  x:'<svg viewBox="0 0 24 24" fill="currentColor"><path d="M12 2a10 10 0 1 0 0 20 10 10 0 0 0 0-20Zm3.5 12.1-1.4 1.4L12 13.4l-2.1 2.1-1.4-1.4L10.6 12 8.5 9.9l1.4-1.4L12 10.6l2.1-2.1 1.4 1.4L13.4 12l2.1 2.1Z"/></svg>',
  cl:'<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="m15 18-6-6 6-6"/></svg>',
  cr:'<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="m9 18 6-6-6-6"/></svg>',
  csl:'<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="m11 17-5-5 5-5"/><path d="m18 17-5-5 5-5"/></svg>',
  csr:'<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="m13 17 5-5-5-5"/><path d="m6 17 5-5-5-5"/></svg>',
  sun:'<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><circle cx="12" cy="12" r="4"/><path d="M12 2v2M12 20v2M2 12h2M20 12h2M5 5l1.5 1.5M17.5 17.5 19 19M19 5l-1.5 1.5M6.5 17.5 5 19"/></svg>',
  moon:'<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2"><path d="M21 12.8A9 9 0 1 1 11.2 3a7 7 0 0 0 9.8 9.8Z"/></svg>'
};

/* ═══════════════ MCP fetch normalizer (handles both result shapes) ═══════════════ */
async function call(name,args){
  const r=await window.cowork.callMcpTool(name,args);
  if(r&&r.isError) throw new Error("Tool error: "+name);
  let d=(r&&r.structuredContent!=null)?r.structuredContent:null;
  if(d==null&&r&&r.content&&r.content[0]&&r.content[0].text){try{d=JSON.parse(r.content[0].text);}catch(e){d=r.content[0].text;}}
  return d;
}

let chartArea=null, chartBars=null, STATE={}, page=0, pageSize=10, curTab="records", curRange=90;

function scard(desc,title,bdg,l1,l1i,l2){
  return `<div class="card scard">
    <div class="hd"><div class="desc">${desc}</div><div class="title">${title}</div><div class="act">${bdg}</div></div>
    <div class="ft"><div class="l1">${l1}${l1i||""}</div><div class="l2">${l2}</div></div>
  </div>`;
}
const badge=(cls,icon,txt)=>'<span class="badge '+cls+'">'+(icon||"")+esc(txt)+'</span>';

async function load(){
  if(chartArea){try{chartArea.destroy()}catch(e){}chartArea=null;}
  if(chartBars){try{chartBars.destroy()}catch(e){}chartBars=null;}
  const app=document.getElementById("app");
  try{
    /* ─── In a real build, replace this block with live calls, e.g.:
         const raw = await call(SOURCE_A, {projectId: PROJECT_ID});
         …and derive `data` from raw. Wrap each OPTIONAL source in its own try/catch.
       For the scaffold we use SAMPLE so it renders before it's wired. ─── */
    const live = (typeof window!=="undefined") && window.cowork && window.cowork.callMcpTool;
    const data = SAMPLE; // ← replace with values derived from call() results

    STATE = {
      records: data.records,
      items: data.items.slice().sort((a,b)=>b.count-a.count)
    };

    document.getElementById("hdrStatus").outerHTML =
      badge("outline", live?SVG.check:"", (live?"Live":"Sample data")) ; // honest about source

    app.innerHTML=`
      <div class="section-cards">
        ${scard("Revenue", money(data.stats.revenue), badge("outline",SVG.up,"+12%"), "Up from last period", SVG.up, "Sum of paid orders")}
        ${scard("Active Users", num(data.stats.users), badge("outline",SVG.up,"+4%"), "Steady growth", SVG.up, "Distinct users this period")}
        ${scard("Records", num(data.stats.records), badge("outline","","Live"), "Total tracked", "", "Across all tables")}
        ${scard("Success Rate", data.stats.success+"%", badge("outline",SVG.up,"Healthy"), "All systems green", SVG.up, "Recent jobs succeeded")}
      </div>

      <div class="block" id="chart">
        <div class="chart-grid">
          <div class="card chart-card">
            <div class="chart-head">
              <div><div class="ct">Activity</div><div class="cs">Two series over the selected period</div></div>
              <div class="tgroup" id="rangeToggle">
                <button data-r="90" class="active">3 months</button>
                <button data-r="30">30 days</button>
                <button data-r="7">7 days</button>
              </div>
            </div>
            <div class="chart-content"><div class="chart-box"><canvas id="area"></canvas></div></div>
          </div>
          <div class="card chart-card">
            <div class="chart-head"><div><div class="ct">Breakdown</div><div class="cs">By category</div></div></div>
            <div class="chart-content"><div class="chart-box"><canvas id="bars"></canvas></div></div>
          </div>
        </div>
      </div>

      <div class="dt" id="table">
        <div class="dt-top">
          <div class="tabslist" id="tabs">
            <button data-tab="records" class="active">Records <span class="b">${STATE.records.length}</span></button>
            <button data-tab="items">Items <span class="b">${STATE.items.length}</span></button>
          </div>
        </div>
        <div class="tbl-wrap" id="tblWrap"></div>
        <div class="dt-foot">
          <div class="sel" id="selInfo"></div>
          <div class="grp">
            <div class="rpp">Rows per page <select class="sel-sm" id="rpp"><option>10</option><option>20</option><option>50</option></select></div>
            <div class="pageinfo" id="pageInfo"></div>
            <div class="pager">
              <button class="pgbtn" id="pgFirst">${SVG.csl}</button>
              <button class="pgbtn" id="pgPrev">${SVG.cl}</button>
              <button class="pgbtn" id="pgNext">${SVG.cr}</button>
              <button class="pgbtn" id="pgLast">${SVG.csr}</button>
            </div>
          </div>
        </div>
      </div>

      <div class="footnote">Sources: describe the real connectors here. Replace SAMPLE data before shipping.</div>
    `;

    drawArea(data.series, curRange);
    drawBars(data.breakdown);

    document.querySelectorAll("#rangeToggle button").forEach(b=>b.addEventListener("click",()=>{
      document.querySelectorAll("#rangeToggle button").forEach(x=>x.classList.remove("active"));b.classList.add("active");
      curRange=+b.dataset.r; drawArea(data.series, curRange);}));

    /* table */
    const HEADERS={records:["Name","Status","Owner","Updated"],items:["Item","Count","Schema"]};
    function statusBadge(s){
      if(s==="READY")return '<span class="badge muted statusb"><span class="ok">'+SVG.check+'</span>Ready</span>';
      if(s==="ERROR")return '<span class="badge muted statusb"><span class="er">'+SVG.x+'</span>Failed</span>';
      return '<span class="badge muted statusb"><span class="ld">'+SVG.loader+'</span>'+esc(s)+'</span>';
    }
    function rowsFor(tab){
      if(tab==="records")return STATE.records.map(r=>({cells:[
        '<span>'+esc(r.name)+'</span>', statusBadge(r.status), esc(r.owner), '<span class="mut">'+fmtDate(r.when)+'</span>'
      ]}));
      return STATE.items.map(it=>({cells:[
        '<span>'+esc(it.label)+'</span>',
        '<span style="font-variant-numeric:tabular-nums">'+num(it.count)+'</span>',
        '<span class="mut">'+esc(it.note)+'</span>'
      ]}));
    }
    function renderTable(){
      const rows=rowsFor(curTab), total=rows.length;
      const cols=HEADERS[curTab].length+1; // +1 for the checkbox column
      const pages=Math.max(1,Math.ceil(total/pageSize));
      if(page>=pages)page=pages-1;if(page<0)page=0;
      const slice=rows.slice(page*pageSize,page*pageSize+pageSize);
      const head='<tr><th class="mid"><input type="checkbox" id="all"></th>'+HEADERS[curTab].map(h=>'<th>'+h+'</th>').join('')+'</tr>';
      const body=slice.map(r=>'<tr><td class="mid"><input type="checkbox" class="rc"></td>'+r.cells.map(c=>'<td>'+c+'</td>').join('')+'</tr>').join('');
      document.getElementById("tblWrap").innerHTML='<table><thead>'+head+'</thead><tbody>'+(body||'<tr><td colspan="'+cols+'" style="text-align:center;padding:36px" class="mut">No results.</td></tr>')+'</tbody></table>';
      document.getElementById("pageInfo").textContent="Page "+(page+1)+" of "+pages;
      const upd=()=>{const sel=document.querySelectorAll("#tblWrap .rc:checked").length;document.getElementById("selInfo").textContent=sel+" of "+total+" row(s) selected.";};
      upd();
      const all=document.getElementById("all");
      if(all)all.addEventListener("change",()=>{document.querySelectorAll("#tblWrap .rc").forEach(c=>c.checked=all.checked);upd();});
      document.querySelectorAll("#tblWrap .rc").forEach(c=>c.addEventListener("change",upd));
      document.getElementById("pgFirst").disabled=page===0;document.getElementById("pgPrev").disabled=page===0;
      document.getElementById("pgNext").disabled=page>=pages-1;document.getElementById("pgLast").disabled=page>=pages-1;
    }
    function setTab(t){curTab=t;page=0;document.querySelectorAll("#tabs button").forEach(b=>b.classList.toggle("active",b.dataset.tab===t));renderTable();}
    document.querySelectorAll("#tabs button").forEach(b=>b.addEventListener("click",()=>setTab(b.dataset.tab)));
    document.getElementById("rpp").addEventListener("change",e=>{pageSize=+e.target.value;page=0;renderTable();});
    document.getElementById("pgFirst").addEventListener("click",()=>{page=0;renderTable();});
    document.getElementById("pgPrev").addEventListener("click",()=>{page--;renderTable();});
    document.getElementById("pgNext").addEventListener("click",()=>{page++;renderTable();});
    document.getElementById("pgLast").addEventListener("click",()=>{page=1e9;renderTable();});
    setTab("records");

    /* sidebar nav (scroll-to) */
    document.querySelectorAll(".smenu-btn[data-go]").forEach(el=>el.addEventListener("click",()=>{
      document.querySelectorAll(".smenu-btn").forEach(x=>x.classList.remove("active"));el.classList.add("active");
      const t=document.getElementById(el.dataset.go);if(t)t.scrollIntoView({behavior:"smooth",block:"start"});}));
    document.getElementById("sbToggle").addEventListener("click",()=>{const s=document.querySelector(".sidebar");s.style.display=s.style.display==="none"?"":"none";});

  }catch(e){
    app.innerHTML='<div class="errbox">Couldn\'t load live data: '+esc(e.message||String(e))+'. Try Reload.</div>';
  }
}

/* ═══════════════ charts read all colors from CSS vars → theme-agnostic ═══════════════ */
function drawArea(series, days){
  if(chartArea){try{chartArea.destroy()}catch(e){}}
  const since=Date.now()-days*864e5, byWeek=days>30;
  const within=series.filter(d=>d.t>=since);
  const map={};within.forEach(d=>{const k=byWeek?weekKey(d.t):dayKey(d.t);map[k]=map[k]||{p:0,s:0};map[k].p+=d.primary;map[k].s+=d.secondary;});
  const labels=[],p=[],s=[];let t=byWeek?weekKey(since):dayKey(since);const end=Date.now();
  // step by re-snapping (not a fixed +ms) so a DST shift can't knock buckets off their keys
  while(t<=end){labels.push(new Date(t).toLocaleDateString("en-US",{month:"short",day:"numeric"}));p.push(map[t]?map[t].p:0);s.push(map[t]?map[t].s:0);t=byWeek?weekKey(t+7*864e5+432e5):dayKey(t+864e5+432e5);}
  const c1=cssVar("--chart-1"), c2=cssVar("--chart-2");
  const grad=(ctx,hex,a0,a1)=>{const g=ctx.createLinearGradient(0,0,0,250);g.addColorStop(0,hexToRgba(hex,a0));g.addColorStop(1,hexToRgba(hex,a1));return g;};
  chartArea=new Chart(document.getElementById("area"),{type:"line",
    data:{labels,datasets:[
      {label:"Secondary",data:s,stack:"a",fill:true,tension:.4,pointRadius:0,borderWidth:1.5,borderColor:c2,backgroundColor:ctx=>grad(ctx.chart.ctx,c2,.35,.02)},
      {label:"Primary",data:p,stack:"a",fill:true,tension:.4,pointRadius:0,borderWidth:1.5,borderColor:c1,backgroundColor:ctx=>grad(ctx.chart.ctx,c1,.5,.03)}
    ]},
    options:{responsive:true,maintainAspectRatio:false,interaction:{mode:"index",intersect:false},
      plugins:{legend:{display:false},tooltip:{backgroundColor:cssVar("--card"),titleColor:cssVar("--foreground"),bodyColor:cssVar("--muted-foreground"),borderColor:cssVar("--border"),borderWidth:1,padding:10,cornerRadius:8,displayColors:true,boxWidth:8,boxHeight:8,usePointStyle:true}},
      scales:{x:{grid:{display:false},border:{display:false},ticks:{color:cssVar("--muted-foreground"),font:{size:11},maxRotation:0,autoSkip:true,maxTicksLimit:7,padding:8}},
        y:{display:false,beginAtZero:true,grid:{color:cssVar("--border"),drawTicks:false},border:{display:false}}}}});
}
function drawBars(breakdown){
  if(chartBars){try{chartBars.destroy()}catch(e){}}
  chartBars=new Chart(document.getElementById("bars"),{type:"bar",
    data:{labels:breakdown.map(b=>b.label),datasets:[{data:breakdown.map(b=>b.value),backgroundColor:cssVar("--chart-1"),borderRadius:5,maxBarThickness:24}]},
    options:{indexAxis:"y",responsive:true,maintainAspectRatio:false,
      plugins:{legend:{display:false},tooltip:{backgroundColor:cssVar("--card"),titleColor:cssVar("--foreground"),bodyColor:cssVar("--muted-foreground"),borderColor:cssVar("--border"),borderWidth:1,padding:10,cornerRadius:8,displayColors:false}},
      scales:{x:{display:false,beginAtZero:true,grid:{display:false},border:{display:false}},
        y:{grid:{display:false},border:{display:false},ticks:{color:cssVar("--muted-foreground"),font:{size:12}}}}}});
}

/* ═══════════════ theme toggle — re-reads CSS vars into the charts on switch ═══════════════ */
function syncThemeIcon(){document.getElementById("themeToggle").innerHTML=document.documentElement.classList.contains("dark")?SVG.sun:SVG.moon;}
document.getElementById("themeToggle").addEventListener("click",()=>{
  const dark=document.documentElement.classList.toggle("dark");
  try{localStorage.setItem("dashkit_theme",dark?"dark":"light");}catch(e){}
  syncThemeIcon();
  // charts hold resolved colors, so redraw them from the new vars
  if(STATE&&STATE.records){drawArea(SAMPLE.series,curRange);drawBars(SAMPLE.breakdown);}
});
syncThemeIcon();
load();
</script>
</body>
</html>
```

## Reference files in this folder (optional — SKILL.md alone is sufficient)
- **`dashboard-template.html`** — the Build kit above as a standalone file you can open and edit
  directly. Identical content.
- **`DASHBOARD-PLAYBOOK.md`** — extended reference: per-source data-shape tables, the
  remote-vs-local deep dive, the `cssVar`/`hexToRgba` rationale, and the mock-preview method.

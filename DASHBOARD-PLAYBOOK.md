# Live Dashboard Playbook (shadcn/ui + agent runtime)

Technical reference for the **dashkit** skill: the verbatim code, tokens, and tables you copy
when building a **self-contained, single-file HTML dashboard that pulls live data through the
Claude/cowork agent runtime** and looks like a genuine shadcn/ui dashboard. The procedure lives
in `SKILL.md`; this file is the "open it when you need the exact code" companion. The clean
scaffold to start from is `dashboard-template.html`.

---

## 1. What we're building

A **single `.html` file** — inline CSS + inline JS, one external `<script>` (Chart.js
from a CDN). No build step, no bundler, no framework. Open in an editor, edit directly.

It is a faithful clone of the shadcn/ui **dashboard-01** block (`ui.shadcn.com`,
`github.com/shadcn-ui/ui`): inset sidebar, a row of stat cards, one or two charts, and a tabbed
paginated data table. The difference from the static shadcn demo: **the numbers are real**,
fetched at load time from the user's own services. The real React components and Recharts can't
run without a build step, so the look is reproduced in CSS + Chart.js — done right it's
indistinguishable (see §3 for the fidelity tells).

Reference sources:
- Components: https://ui.shadcn.com/docs/components
- Source / blocks: https://github.com/shadcn-ui/ui (see `blocks/dashboard-01`)

---

## 2. The runtime contract (the part that's easy to miss)

The dashboard does **not** use `fetch()` or static data. It calls MCP tools through a
global the agent host injects:

```js
const r = await window.cowork.callMcpTool(toolName, args);
```

`window.cowork` only exists **inside the Claude/cowork agent runtime**. If you open the
file in a normal browser by double-clicking, `window.cowork` is `undefined`, every call
throws, and the dashboard shows its error box ("Couldn't load live data… Try Reload").
That is expected, not a bug. (If the bridge fails *inside* the runtime with a remote connector
wired, confirm the global is still named `window.cowork.callMcpTool` — runtime APIs drift.)

### ⚠️ Connectors must be REMOTE, not local — the #1 setup-loop trap

The dashboard runs as a **Claude Artifact: a sandboxed iframe**. The only way data reaches
it is the `window.cowork.callMcpTool` bridge, and that bridge can only reach **remote MCP
connectors** — the URL/OAuth connectors you enable in Claude → Settings → Connectors. It
**cannot** reach a **local / stdio MCP server** (the kind configured in Claude Desktop's
config or a Claude Code `.mcp.json`): the artifact sandbox has no way to spawn or talk to a
local process. There is no bridge out of the sandbox to your machine.

This is the loop people fall into: they have, say, a local Postgres MCP, the agent wires
the dashboard to it, the artifact loads, every `callMcpTool` silently fails, and the user
just sees the red "Couldn't load live data" box with no clue why. **Check connectivity up
front:** confirm the data source is a *remote* connector before building. If the only thing
available is a local MCP, stop and tell the user to add a remote connector (or a hosted
equivalent) first — don't build a dashboard that can't reach its data.

### Discovering what's actually connected (don't hardcode blindly)

The building agent (Claude, in the conversation) already has the connected remote MCP tools
in **its own tool list** — that's the source of truth, not a guess. Procedure:

1. Look at which MCP tools you (the agent) can call this session. Their names are the exact
   `mcp__<server-uuid>__<operation>` strings the artifact will use — copy them verbatim.
2. If a tool needs IDs/params you don't have (a Vercel `projectId`, a Supabase `project_ref`),
   call the relevant **list** tool first (`list_projects`, `list_tables`, …) or ask the user
   — don't invent IDs.
3. Make one cheap probe call per source to confirm it returns data before committing to a
   layout. A source that errors on probe is one you design *around*, not into.

### The `call()` helper (copy this verbatim)

MCP results arrive in one of two shapes. Always normalize:

```js
async function call(name, args){
  const r = await window.cowork.callMcpTool(name, args);
  if (r && r.isError) throw new Error("Tool error: " + name);
  let d = (r && r.structuredContent != null) ? r.structuredContent : null;
  if (d == null && r && r.content && r.content[0] && r.content[0].text){
    try { d = JSON.parse(r.content[0].text); } catch(e){ d = r.content[0].text; }
  }
  return d;
}
```

### Tool names are project-specific MCP server IDs

They look like `mcp__<server-uuid>__<operation>`. Pin them in consts at the top of the
script so they're easy to swap:

```js
const VERCEL = "mcp__<uuid>__list_deployments";
const SUPA   = "mcp__<uuid>__list_tables";
const STRIPE = "mcp__<uuid>__stripe_api_read";
```

The server UUIDs differ per user/session — get them from the discovery step above, never
from this doc. The Vercel/Supabase/Stripe trio below is just one example set; a real
build uses whatever remote connectors *this* user has (Linear, GitHub, Notion, Postgres,
Google Sheets, …).

### Data shapes for a typical multi-source build (handy reference)

| Source | Tool / op | Args | Returns (after `call()`) |
|---|---|---|---|
| Vercel | `list_deployments` | `{projectId, teamId}` | `{deployments:{deployments:[…]}}` or `{deployments:[…]}`; each dep `{created(ms), state:"READY"|"ERROR"|…, url, meta:{githubCommitMessage, githubCommitRef, githubCommitAuthorName}}` |
| Supabase | `list_tables` | `{project_id, schemas:["public"], verbose:false}` | `{tables:[{name:"public.x", rows, rls_enabled}]}` |
| Stripe | `stripe_api_read` | `{stripe_api_operation_id:"GetBalance"\|"GetCharges"\|"GetPrices"\|"GetProducts", parameters:{…}}` | balance `{available:[{amount}],pending:[{amount}]}`; charges/prices/products `{data:[…]}` (amounts in **cents**, divide by 100) |

Run independent calls with `Promise.all`. Wrap each **optional** source (e.g. Stripe) in
its own `try/catch` so one dead integration degrades gracefully instead of blanking the
whole page (pattern: `Net Revenue` shows "Pre-revenue" instead of erroring).

---

## 3. Design system: shadcn/ui tokens (light + dark)

Drive **everything** off CSS custom properties so the theme is one class on `<html>`. Put the
**light** tokens on `:root` and the **dark** overrides under `.dark` — the shadcn convention.
Use the **Neutral** palette, `new-york` style, radius 10px. Default to dark (set the `.dark`
class before first paint to avoid a flash; see the toggle in §4 and the template).

```css
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
  --chart-1:#e76e50; --chart-2:#2a9d90; --chart-3:#274754; --chart-4:#e8c468; --chart-5:#f4a462;
  --scard-top:rgba(10,10,10,.04);   /* card top-gradient = primary/5 */
  --row-hover:rgba(10,10,10,.03);
  --radius:10px; --radius-lg:10px; --radius-md:8px; --radius-sm:6px;
  --shadow-sm:0 1px 3px 0 rgba(0,0,0,.1),0 1px 2px -1px rgba(0,0,0,.1);
  --shadow-xs:0 1px 2px 0 rgba(0,0,0,.05);
}
.dark{
  color-scheme:dark;                                   /* makes native checkbox/select dark */
  --background:#0a0a0a; --foreground:#fafafa;
  --card:#1c1c1c; --card-foreground:#fafafa;           /* cards are LIGHTER than bg = elevated */
  --muted:#272727; --muted-foreground:#a1a1a1;
  --border:#2a2a2a; --input:#2f2f2f; --ring:#8a8a8a;
  --primary:#fafafa; --primary-foreground:#0a0a0a;     /* primary INVERTS in dark mode */
  --secondary:#272727; --accent:#272727; --accent-foreground:#fafafa;
  --destructive:#ff5b5b;                               /* brighter red on dark */
  --sidebar:#1c1c1c; --sidebar-foreground:#fafafa; --sidebar-accent:#2a2a2a;
  --sidebar-border:#2a2a2a; --sidebar-primary:#fafafa;
  --green:#22c55e;
  --chart-1:#2662d9; --chart-2:#2eb88a; --chart-3:#e88c30; --chart-4:#af57db; --chart-5:#e23670;
  --scard-top:rgba(255,255,255,.05);
  --row-hover:rgba(255,255,255,.04);
  --shadow-sm:0 1px 3px 0 rgba(0,0,0,.4),0 1px 2px -1px rgba(0,0,0,.4);   /* stronger in dark or they vanish */
  --shadow-xs:0 1px 2px 0 rgba(0,0,0,.3);
}
```

`--chart-1`…`--chart-5` are shadcn's chart palette — use them for chart series instead of an
arbitrary hex. There's no separate "for reference" theme: both sets are first-class, and **no
color is hardcoded anywhere** (the few semi-transparent accents the CSS needs — the stat-card
top gradient and table-row hover — are tokens too: `--scard-top`, `--row-hover`).

### Typography — Geist if you can load it, graceful fallback if you can't

shadcn/ui ships **Geist**, and a generic `ui-sans-serif,-apple-system,…` stack (San Francisco on
macOS) is a real "not-quite-shadcn" tell. **But the Claude Artifact sandbox has a CSP** that may
block font CDNs, so treat the font as best-effort, not a hard requirement: the tokens, spacing,
and dashboard-01 layout carry ~90% of the look; the font is the last 10%. Don't block on it.

Load Geist from **jsdelivr** — the same origin that serves Chart.js, so it clears the same
allowlist (the `@fontsource` family is `"Geist Sans"`, not `"Geist"`):

```html
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@fontsource/geist-sans@5/400.css">
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@fontsource/geist-sans@5/500.css">
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@fontsource/geist-sans@5/600.css">
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@fontsource/geist-sans@5/700.css">
<!-- Google Fonts alt, if your sandbox allows it (family "Geist"):
     https://fonts.googleapis.com/css2?family=Geist:wght@300..700&display=swap -->
```
```css
body{ font-family:"Geist Sans","Geist",ui-sans-serif,system-ui,-apple-system,BlinkMacSystemFont,"Segoe UI",Roboto,Helvetica,Arial,sans-serif; }
```
```js
if(window.Chart) Chart.defaults.font.family = '"Geist Sans","Geist",ui-sans-serif,system-ui,sans-serif';
```

`font-src`/`style-src` are separate CSP directives from `script-src`, so jsdelivr serving Chart.js
doesn't *guarantee* it'll serve the font — verify in a real artifact. If both CDNs are blocked, the
system fallback applies and that's fine. Stat-card numbers use `font-variant-numeric:tabular-nums`.

### Layout relationships that matter

- `body` / gutter uses `--sidebar`; the main inset panel uses `--background`.
- In **dark**, `--background` (#0a0a0a) is *darker* than `--sidebar`/`--card` (#1c1c1c); the
  inset reads as a floating panel via its **border + shadow**, not by being lighter. In
  **light**, the white inset (`#fff`) sits on a slightly-gray gutter (`#fafafa`) — the panel is
  lighter. Both are exactly what shadcn does; the token swap handles it.
- Shadows must get **stronger** in dark mode (≈0.4 alpha) or they vanish — hence the per-theme
  `--shadow-sm`/`--shadow-xs` above.

---

## 4. Theme-aware charts: read colors from CSS vars (one source of truth)

CSS variables don't reach into Chart.js config or string-built SVG/markup. If you hardcode a
color there (`#fafafa`, `#1c1c1c`, `rgba(10,10,10,…)`), it's frozen to one theme and goes wrong
— invisible, or glaringly wrong — the instant the user toggles. **Never hardcode a color in JS.**
Read the active token at draw time:

```js
const cssVar = n => getComputedStyle(document.documentElement).getPropertyValue(n).trim();
function hexToRgba(hex, a){                      // for gradient fills that need an alpha
  hex = (hex||"").replace("#","");
  if (hex.length === 3) hex = hex.split("").map(c=>c+c).join("");
  const n = parseInt(hex||"000000", 16);
  return "rgba("+((n>>16)&255)+","+((n>>8)&255)+","+(n&255)+","+a+")";
}
```

Every chart color must come from a token. The properties that need one:

| Chart property | Token to read |
|---|---|
| Line/area `borderColor` | `cssVar("--chart-1")`, `cssVar("--chart-2")`, … |
| Area fill gradient | `hexToRgba(cssVar("--chart-1"), 0.5→0.03)` (low alphas = subtle wash, not a block) |
| Bar `backgroundColor` | `cssVar("--chart-1")` (or per-series chart tokens) |
| Tooltip `backgroundColor` | `cssVar("--card")` |
| Tooltip `borderColor` (+ `borderWidth:1`) | `cssVar("--border")` |
| Tooltip `titleColor` / `bodyColor` | `cssVar("--foreground")` / `cssVar("--muted-foreground")` |
| Axis tick `color` | `cssVar("--muted-foreground")` |
| Gridline `color` | `cssVar("--border")` |

**On theme toggle, redraw the charts** so they re-read the vars. Chart.js bakes resolved colors
into the instance at construction, so the toggle handler destroys and rebuilds (or `.update()`s)
each chart:

```js
document.getElementById("themeToggle").addEventListener("click", ()=>{
  const dark = document.documentElement.classList.toggle("dark");
  try { localStorage.setItem("dashkit_theme", dark ? "dark" : "light"); } catch(e){}
  drawArea(/*…*/); drawBars(/*…*/);   // re-read CSS vars into fresh charts
});
```

Using the `--chart-*` tokens means series are colorful and on-brand in both themes; "dark theme"
means dark surfaces, not literally monochrome.

---

## 5. No fake elements — every visible control must do something

The biggest "AI slop" tell in a generated dashboard is decorative chrome that looks
interactive but is dead. Rule: **if it looks clickable, it must work — otherwise remove
it.** Three outcomes per element:

1. **Wire it for real.** Sidebar "Resources" links → real `<a href target="_blank">` to
   the actual dashboards you can construct from known IDs:
   - Vercel team: `https://vercel.com/<team-slug>`
   - Supabase: `https://supabase.com/dashboard/project/<project-ref>`
   - Stripe: `https://dashboard.stripe.com/`
   - Deployment rows → link the commit cell to `https://<deployment.url>`.
2. **Remove it** if it can't be made to mean anything: Quick Create, Inbox, Settings,
   Get Help, Search, "Customize Columns", "Add Section", per-row drag-handles, per-row
   ⋯ menus, logo `href="#"`.
3. **Keep only if it has a visible effect.** Row checkboxes stayed because they update a
   live "N of M selected" count; pagination, tab switching, sidebar scroll-to, the range
   toggle, and the theme toggle all genuinely work.

When you delete a table column (grip/⋯), update the empty-state `colspan` to match the
new column count, or the "No results" row spans wrong. (The template computes it from the
header count, so it stays correct.)

Also strip dead affordances: a footer user-chip that opens no menu should lose its ⋯ icon
and its `:hover` background so it doesn't imply a click target.

---

## 6. Verifying a runtime-coupled dashboard (you can't just open it)

**Which path applies depends on where you're building.**

- **Inside the Claude.ai artifact runtime** (the normal case): `window.cowork` is live, so the
  artifact preview *is* the verification — load it and watch real data populate. Toggle the
  theme and confirm both look right. You can't headlessly screenshot from inside the sandbox;
  rely on the live preview and the on-screen error box. If data doesn't load, re-check the
  remote-vs-local connector rule in §2 first.
- **In a coding agent** (Claude Code, etc., editing the file on disk): `window.cowork` doesn't
  exist, so use the **throwaway mock preview** below to screenshot it headlessly.

Mock-preview path (coding-agent only), then delete it:

1. `cp dashboard.html __preview.html`
2. Inject a `<script>` right after `<title>` that defines `window.cowork.callMcpTool` to
   return representative fake data in the `{structuredContent: …}` shape, branching on the
   tool name (`name.indexOf("list_deployments") > -1`, etc.). Use real `Date.now()` so the
   time-bucketed chart populates. (The template renders from its own `SAMPLE` block without a
   mock, so for the template you can screenshot it directly.)
3. Render + screenshot headlessly. With gstack: `browse goto file://<abs>/__preview.html`
   then `browse screenshot out.png` (and `browse console --errors` to catch JS errors).
   Screenshot **both themes** — toggle via the header button or by adding/removing the `.dark`
   class on `<html>`.
4. Read the PNG, confirm: surfaces correct for the theme, charts/tooltips/gridlines/ticks
   visible, no dead controls, badges colored right. Then `rm __preview.html`.

The preview is byte-identical to the real file except the injected mock, so what you see
is what the runtime will render.

---

## 7. Security / hygiene notes

- A finished dashboard embeds **real identifiers** (Vercel project & team IDs, Supabase project
  ref, MCP server UUIDs, the user's email). Fine for personal use; **scrub or parameterize
  before sharing** the file or committing it anywhere public. (The template and the worked
  example ship with placeholder IDs for exactly this reason.)
- Always HTML-escape any value that comes from live data before injecting into markup
  (`esc()` helper covering `& < > "`). Commit messages, table names, plan names are all
  attacker-influenced in theory.
- Add `rel="noopener"` to every `target="_blank"` link.

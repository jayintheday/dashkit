# dashkit

A Claude skill that **applies [shadcn/ui](https://ui.shadcn.com) components to your live data**
to build a polished, single-file dashboard — delivered as a self-contained Claude Artifact wired
to your own remote MCP connectors, so the numbers are **real, not placeholder**. Light and dark,
dark by default.

It's a **component kit you compose**, not one fixed layout. Ask Claude for a dashboard over your
connected services (Vercel, Supabase, Stripe, Linear, GitHub, Notion, Postgres, Google Sheets, …)
and dashkit composes the right shadcn pieces for *that* data into a single `.html` artifact that
fetches your live data at load time — without the generic "AI slop" look.

## What it produces

- shadcn components as composable pieces: stat cards, an area/line chart, a horizontal bar chart,
  a tabbed paginated table, badges, an inset sidebar, a header with a theme toggle.
- The agent **composes the layout to fit the data** — a status page might be stat cards + one
  table and no charts; a metrics view might be charts and no table. Not every dashboard is the same.
- shadcn **Neutral** tokens (`new-york` style), Geist font, a working light/dark toggle.
- Charts (Chart.js) that read their colors from CSS variables, so they follow the theme.
- **No fake elements** — every visible control works or is removed.

Open [`dashboard-template.html`](dashboard-template.html) to see the kit composed into one example
layout, in both themes.

## Install

**Claude Code** — clone into your skills directory:

```bash
git clone https://github.com/jayintheday/dashkit ~/.claude/skills/dashkit
```

Then invoke it with `/dashkit`, or just ask Claude to "build a dashboard for my connected
services".

**claude.ai** — upload `SKILL.md` as a custom skill/capability. The skill is **self-sufficient
in `SKILL.md` alone** (the full component kit is embedded under its "Build kit" section), so the
single file is all you need there.

## How it works

The dashboard runs as a Claude Artifact (a sandboxed iframe). It does **not** use `fetch()`;
data reaches it only through the runtime bridge `window.cowork.callMcpTool(name, args)`, which
can reach **remote MCP connectors only** (the URL/OAuth ones under Claude → Settings →
Connectors) — not a local/stdio MCP server. dashkit checks this up front and follows a
five-phase procedure: **discover → review → design the layout → compose → verify.**

## Requirements

- A Claude environment with Artifacts and at least one **remote** MCP connector for your data.
- That's it — the artifact is a single HTML file; its only external dependencies are the
  Chart.js and Geist CDN tags.

## Repo contents

| File | Purpose |
|---|---|
| `SKILL.md` | The skill. Self-sufficient — embeds the full component kit ("Build kit") + the design rules, procedure, and data-shape reference. |
| `dashboard-template.html` | The Build kit as a standalone, openable file (the Foundation composed into one example layout). **Identical to the block embedded in `SKILL.md` — keep the two in sync when editing.** |

## Contributing / developing

Edit `SKILL.md` (and, if you change the component kit, mirror it in `dashboard-template.html` so
they stay byte-identical), then commit and push. Issues and PRs welcome.

## License

[MIT](LICENSE) © 2026 Vijay Patel

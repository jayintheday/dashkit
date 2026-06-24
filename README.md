# dashkit

A Claude skill that builds **live, single-file dashboards** styled like the
[shadcn/ui](https://ui.shadcn.com) **dashboard-01** block — delivered as a self-contained
Claude Artifact wired to your own remote MCP connectors, so the numbers are **real, not
placeholder**. Light and dark themes, dark by default.

Ask Claude for a dashboard over your connected services (Vercel, Supabase, Stripe, Linear,
GitHub, Notion, Postgres, Google Sheets, …) and dashkit produces a polished, single `.html`
artifact that fetches your live data at load time — without the generic "AI slop" look.

## What it produces

- The shadcn dashboard-01 shell: inset sidebar, a row of stat cards, 1–2 charts, a tabbed
  paginated table.
- shadcn **Neutral** tokens (`new-york` style), Geist font, a working light/dark toggle.
- Charts (Chart.js) that read their colors from CSS variables, so they follow the theme.
- **No fake elements** — every visible control works or is removed.

Open [`dashboard-template.html`](dashboard-template.html) to view the clean scaffold in both
themes.

## Install

**Claude Code** — clone into your skills directory:

```bash
git clone https://github.com/jayintheday/dashkit ~/.claude/skills/dashkit
```

Then invoke it with `/dashkit`, or just ask Claude to "build a dashboard for my connected
services".

**claude.ai** — upload `SKILL.md` as a custom skill/capability. The skill is **self-sufficient
in `SKILL.md` alone** (the full styled scaffold is embedded under its "Build kit" section), so
the single file is all you need there.

## How it works

The dashboard runs as a Claude Artifact (a sandboxed iframe). It does **not** use `fetch()`;
data reaches it only through the runtime bridge `window.cowork.callMcpTool(name, args)`, which
can reach **remote MCP connectors only** (the URL/OAuth ones under Claude → Settings →
Connectors) — not a local/stdio MCP server. dashkit checks this up front and follows a
five-phase procedure: **discover → review → propose → build → verify.**

## Requirements

- A Claude environment with Artifacts and at least one **remote** MCP connector for your data.
- That's it — the artifact is a single HTML file; its only external dependencies are the
  Chart.js and Geist CDN tags.

## Repo contents

| File | Purpose |
|---|---|
| `SKILL.md` | The skill. Self-sufficient — embeds the complete styled scaffold ("Build kit"). |
| `dashboard-template.html` | The Build kit as a standalone, openable file. **Identical to the block embedded in `SKILL.md` — keep the two in sync when editing.** |
| `DASHBOARD-PLAYBOOK.md` | Extended reference: data-shape tables, the remote-vs-local deep dive, the theme-aware chart pattern, the mock-preview method. |

## Contributing / developing

Edit `SKILL.md` (and, if you change the scaffold, mirror it in `dashboard-template.html` so
they stay byte-identical), then commit and push. Issues and PRs welcome.

## License

[MIT](LICENSE) © 2026 Vijay Patel

# AgentsPlugin

Convenience package so Claude, Cursor, and ChatGPT can install RealtyPad **skills** and connect the hosted **MCP** tools in one step.

Plugin id is `realtypad` (lowercase, required by Agent Plugins / Claude / Cursor). In Claude Code that installs as `/plugin install realtypad@realtypad`.

**This repo is the source of truth for RealtyPad agent playbooks.** Each workflow is a full [Agent Skill](https://agentskills.io/): `SKILL.md` holds the procedure and hard rules, and long detail lives in that skill's `references/`. The same files serve external agents (Claude, Cursor, ChatGPT/Codex) and RealtyPad's hosted automation worker. RealtyPad docs (`search_docs` / `read_doc` on the MCP) are supplementary reference, not a second playbook.

MCP URL (no trailing slash): `https://app.realtypad.ai/mcp`  
Stage: `https://stage.realtypad.ai/mcp`

Auth is browser OAuth in the client. This repo does not ship secrets.

## Install

### One command (Claude Code, Cursor, Codex, and others)

```bash
npx plugins add RealtyPad/AgentsPlugin
```

The [plugins CLI](https://github.com/vercel-labs/plugins) detects installed agents and maps this package into each native plugin layout.

### Cursor

This repo is an [Agent Plugin](https://agent-plugins.org/) (`plugin.json` + `mcp.json` + `skills/`) and also includes `.cursor-plugin/plugin.json` for Cursor’s plugin format.

- Marketplace / Customize: add the Git repo once it is listed or linked.
- Local: clone into `~/.cursor/plugins/local/realtypad` (or use `npx plugins add .` from a checkout).

After install, complete OAuth for the RealtyPad MCP server, then ask the agent to call `list_deals` (limit 3) and load the `realtypad` skill.

### Claude Code

```text
/plugin marketplace add RealtyPad/AgentsPlugin
/plugin install realtypad@realtypad
```

Or load a checkout:

```bash
claude --plugin-dir /path/to/AgentsPlugin
```

Claude Code reads `.claude-plugin/plugin.json` and `.mcp.json`. Same skills folder as the other clients.

### ChatGPT

ChatGPT needs the **public HTTPS** MCP URL. Localhost will not work.

**Connector (works today):** Settings → Apps → Developer mode → add connector → `https://app.realtypad.ai/mcp` → OAuth.

**Plugin (skills + MCP together):** this package also has `.codex-plugin/plugin.json` for ChatGPT/Codex. Public directory listing is review-gated; until then, use the connector plus these skills if the plugin is installed from a local/workspace source.

## Layout

```text
plugin.json                 Agent Plugins 1.0 (Cursor, ChatGPT/Codex, Copilot, VS Code)
mcp.json                    Agent Plugins MCP (streamable-http)
.mcp.json                   Claude Code / Codex HTTP MCP
.claude-plugin/             Claude Code manifest + marketplace catalog
.cursor-plugin/             Cursor plugin manifest
.codex-plugin/              ChatGPT / Codex plugin manifest
skills/                     Shared Agent Skills
```

## Skills

| Skill | When |
|-------|------|
| `realtypad` | Session start: mission, pipeline, status tree, write rules, skill index; `references/glossary.md` |
| `ingest` | Sweeps, scrapes, listing upsert; `references/data-sources.md` + per-portal recipes |
| `research` | Tax/rent/ARV/photos/comps/listing history; no ranking |
| `growth-drivers` | Qualitative demand signals (employers, transit, rezoning) → Drivers |
| `triage` | Queue batches: new, Blocking/Soft gaps, blocked/watch/ranked |
| `scenarios` | Persist financing × strategy runs (Preview ≠ thesis) |
| `underwrite` | Verdict, `advance_deal`, buyer-fit gate |
| `trends` | `ensure_geo_markets` + `refresh_deal_trends(gap_fill=true)`; surgical catalog refresh |
| `buyers` | Buyer book, shares, share-thread chat |
| `distress` | Auction / REO / assign-to-flipper |
| `redevelopment` | Land / tear-down / densification residual |
| `cost-estimate` | CapEx / Fix-up costs (repair items) |
| `opex` | OpEx / Monthly costs (opex items) |

### Editing playbooks

Edit procedures here, not in RealtyPad `docs/user/agents/`. Keep content model-neutral (the hosted worker is not Claude) and keep each `SKILL.md` under ~500 lines — move long detail into `references/`. Cross-skill links use relative paths (`../ingest/references/data-sources/zillow.md`). RealtyPad's API pins this repo by release tag, so a playbook change reaches hosted automation only after a tag and a pin bump.

## Local MCP

For operator machines hitting a local API instead of production, override the client MCP config (do not commit tokens):

```json
{
  "mcpServers": {
    "realtypad": {
      "url": "http://localhost:8000/mcp"
    }
  }
}
```

Cursor cloud OAuth against localhost often needs `mcp-remote`. See RealtyPad `docs/user/cursor-mcp.md`.

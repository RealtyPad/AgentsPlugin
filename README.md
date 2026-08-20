# AgentsPlugin

Convenience package so Claude, Cursor, and ChatGPT can install RealtyPad **skills** and connect the hosted **MCP** tools in one step.

Plugin id is `realtypad` (lowercase, required by Agent Plugins / Claude / Cursor). In Claude Code that installs as `/plugin install realtypad@realtypad`.

Live operating procedures stay on the server. At session start, agents should call MCP tool `get_agent_manual`. Skills in this repo are install-time triggers and short checklists — not a second playbook.

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

After install, complete OAuth for the RealtyPad MCP server, then ask the agent to call `list_deals` (limit 3) and `get_agent_manual`.

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

**Plugin (skills + MCP together):** this package also has `.codex-plugin/plugin.json` for ChatGPT/Codex. Public directory listing is review-gated; until then, use the connector and rely on `get_agent_manual` plus these skills if the plugin is installed from a local/workspace source.

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
| `realtypad` | Session start; load the full manual |
| `ingest` | Sweeps, scrapes, listing upsert |
| `research` | Tax/rent/ARV/photos/comps; no ranking |
| `triage` | Queue batches, needs-input, shortlist |
| `scenarios` | Persist financing × strategy runs |
| `underwrite` | Verdict, status, buyer-fit gate |
| `trends` | Catalog / market snapshot refresh |
| `buyers` | Buyer book, shares, share-thread chat |
| `distress` | Auction / REO / assign-to-flipper |
| `cost-estimate` | CapEx / repair BOM (`repairs` alias) |

Canonical workflow text is assembled on the API from HouseMaxxing `api/app/mcp_guides/skills/` and served by `get_agent_manual`. Edit procedures there; keep this plugin’s skills short.

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

Cursor cloud OAuth against localhost often needs `mcp-remote`. See HouseMaxxing `docs/user/cursor-mcp.md`.

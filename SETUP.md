# Setup

This plugin connects Claude to RealtyPad's hosted MCP server at `https://app.realtypad.ai/mcp`. Authentication is OAuth in the client; this repo ships no credentials, tokens, or headers.

## Prerequisites

- A RealtyPad account (sign up at [realtypad.ai](https://realtypad.ai) if needed).
- Claude Code or Cowork with plugin support. For Claude Chat alone, add the MCP connector directly: Settings → Apps → Developer mode → `https://app.realtypad.ai/mcp`.

## Claude Code

1. Install the plugin (see [README](README.md#install)):
   ```text
   /plugin marketplace add RealtyPad/AgentsPlugin
   /plugin install realtypad@realtypad
   ```
2. Run `/mcp` inside Claude Code, select the `realtypad` server, and authenticate. Claude Code opens RealtyPad OAuth in your browser.
3. Sign in and complete the consent flow. All MCP tools are tenant-scoped to the account you authenticate with.

## Cowork

1. Install the plugin from the Plugins page (see [README](README.md#install)).
2. When prompted for connector authentication during install, complete RealtyPad OAuth in the browser.
3. Manage the connector from the installed plugin's connector page under Customize → Plugins.

## Verify the connection

Ask Claude to:

1. Call `list_deals` with `limit: 3` — confirms MCP connectivity and returns deals for your tenant.
2. Call `get_agent_manual` — loads the full operating manual from the server.

If either call fails with auth or workspace errors, reconnect (below) before running ingest, research, triage, or underwrite workflows.

## Tenant and workspace

Your workspace is determined entirely by the OAuth session. Claude cannot infer or switch tenants from your working directory, repository, or chat context. If tools return the wrong deals or a `tenant_id_fkey` error, disconnect and re-authenticate with the correct account.

## Reconnect and disconnect

- **Claude Code:** run `/mcp`, select `realtypad`, and use the authentication options to re-authenticate or clear authentication. Uninstalling the plugin removes the server configuration.
- **Cowork:** sign out or back in from the plugin's connector page; uninstall the plugin to remove the connector.

## Stage and local development

- **Stage MCP:** `https://stage.realtypad.ai/mcp` (same OAuth flow; use only when explicitly testing against stage).
- **Local API:** override the client MCP URL to `http://localhost:8000/mcp`. Do not commit tokens. Cursor cloud against localhost may need `mcp-remote`; see HouseMaxxing `docs/user/cursor-mcp.md`.

After any URL override, re-run OAuth if the client prompts for it.

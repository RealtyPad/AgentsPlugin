---
name: realtypad
description: >-
  Load RealtyPad operating procedures and connect the hosted MCP. Use at session
  start, when the user mentions RealtyPad, deals, underwriting, triage, ingest,
  buyers, distress, CapEx, repairs, or when MCP tools are available but workflow
  rules are needed.
---

# RealtyPad

This plugin ships these skills plus MCP tools at `https://app.realtypad.ai/mcp`.
Canonical procedures live on the server. Do not invent a parallel playbook.

## Session start

1. Confirm the RealtyPad MCP server is connected (OAuth in the client; no secrets in this repo).
2. Call `get_agent_manual` with `workflow` omitted for the full manual.
3. Before a specific task, call `get_agent_manual` again with that workflow key.

Workflow keys: `ingest`, `research`, `triage`, `scenarios`, `underwrite`, `trends`, `buyers`, `distress`, `cost-estimate`. `investors` → `buyers`; `repairs` / `repair-estimate` → `cost-estimate`.

MCP resources and prompts are optional extras. Many clients cannot read them. Prefer `get_agent_manual`.

## Hard rules

- Never invent numbers. Null beats fiction. Mark estimates.
- Pipeline: **ingest (detail page, not search card) → research (+ `ensure_deal_trends` if stale) → triage → scenarios → underwrite** (prefer `advance_deal` when score-ready).
- Deal status: `new` → `researching` → `blocked` \| `watch` \| `ranked` \| `passed`. Human escalate `ranked` → `pursuing` → `closed`. Ingest default is **`new`**. `watch` = Hold — no buyers yet.
- **Writes:** money/research → `update_deal` / money-only `update_deals`. Status + Status note → **`update_deal_status`** (or status-only bulk). Never mix status and money in one call.
- Score glossary: bare `score` = **Economics**; prefer `score_summary` (`economics` / Data confidence / Growth / **Overall**). Read `data_gaps` before advancing.
- CapEx (repair items → `cash_in`) ≠ OpEx (`opex_items` → monthly CF). Never bump `maintenance_pct` as a proxy.
- Do not scrape ToS-hostile sites without explicit user approval.
- All tools are tenant-scoped after OAuth. Reconnect if you see workspace / `tenant_id_fkey` errors.

## Aggressiveness

Defaults by `income_strategy` (chat override `aggressive|moderate|strict`):

| Strategy | Mode | Clears economic bar when |
|----------|------|--------------------------|
| `ltr` / `str` | moderate | Prefer hold projections; fallback CF ≳ ~$100/mo |
| `fix_flip` / `wholesale` | aggressive | Asymmetric only (flip ≳ ~$40–50k; wholesale ≳ ~$25–40k) |

`strict` raises floors (~$200/mo / flip ≥$50k / wholesale ≥$40k).

**Buyer-fit gate:** after economics clear, `match_deal_buyers` — ≥1 match → `ranked`; scanned but none fit → `passed`; empty buyer book → **`watch`**. Hard external gaps → **`blocked`**. Distress: stay `researching`/`blocked` until tax/title/repairs are credible, then apply the gate.

## Quick enums

| Field | Values |
|-------|--------|
| `build_type` | `resale` or `new_build` only — omit if unknown |
| `income_strategy` | `ltr`, `str`, `fix_flip`, `wholesale`; **null** when `deal_purpose=primary` |
| `deal_purpose` | `investment` (default), `primary` (owner-occupied), `either` |
| Deal status | `new`, `researching`, `blocked`, `watch`, `ranked`, `pursuing`, `passed`, `closed` |

Property structure (beds/baths/sqft/year_built/lot_size_acres/address): `update_deal_property`, not `update_deal`.

Prefer bulk writes: `add_manual_leads`, `update_deals` (money-only or status-only — not mixed), `add_deal_attachment_urls`, `find_deal_comps`, `add_deal_comps`, `add_deal_appraisal_tax_history` (max 100).

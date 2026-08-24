---
name: triage
description: >-
  Process the RealtyPad triage queue in batches: new inbox, needs-research,
  needs-verification, blocked/watch holds, ranked/pursuing stages, or regional
  cluster passes. Load deals via list_deals, refresh stale market trends for the
  cluster, call research for data gaps, pattern-pass or underwrite, update
  statuses, and brief. Use when the user asks to triage, clear the queue, new
  inbox, needs research, needs verification, blocked, regional pass, or
  batch-pass a market/builder cluster.
---

# RealtyPad — triage

Call `get_agent_manual(workflow="triage")` before running this workflow. Follow that Markdown.

## Hard rules

- Prefer evidence over score. Scores with estimated tax or capped rent lie.
- Prioritize research on decent Economics + low Confidence (`needs_input`) / Growth gaps, and on **`needs_research`** (missing hard inputs / unscored).
- Fill data gaps with the `research` skill before judging. Hard external gaps → `blocked`.
- Regional/cluster passes: refresh stale markets once via `trends`.
- Individual verdicts: `underwrite` (which calls `scenarios`). Pattern-pass only when the failure mode is already documented.
- Pass/Block require a short rationale note (`rationale` / status note).
- Read comments before flipping a deal that was previously researched.

## Queue modes

Match UI `/triage` stage tabs:

- **new** (default inbox): `list_deals(status="new")` — ingested, not started. Start → `researching`, or pass / block.
- **needs_research**: `list_deals(status="new,researching", needs_research=true)` — missing hard inputs or unscored.
- **needs_verification**: `list_deals(needs_input=true)` — soft CF flags (estimated tax/insurance, rent capped, `repairs_unknown`).
- **blocked / watch / ranked / pursuing**: `list_deals(status=…)` for that stage.
- **cluster**: same market/builder/pattern; refresh trends once, then batch.

## Status quick guide

| Status | When |
|--------|------|
| `new` | Just ingested; not started |
| `researching` | Actively working gaps / UW |
| `blocked` | Hard stop until an external fact clears |
| `watch` | Soft hold (often empty buyer book / timing) |
| `ranked` | Economics clear **and** ≥1 buyer match |
| `pursuing` | Human escalate from ranked |
| `passed` | Below bar, structural blocker, or no buyer fit |

## Aggressiveness

`ltr`/`str` moderate (prefer hold projections; fallback ~$100/mo). `fix_flip`/`wholesale` aggressive (asymmetric only). `strict` raises floors. Pursue (`ranked`) only after `match_deal_buyers` finds ≥1 buyer fit, except structural pattern-passes. Empty buyer book → `watch`; hard external gaps → `blocked`.

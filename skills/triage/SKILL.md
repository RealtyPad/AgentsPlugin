---
name: triage
description: >-
  Process the RealtyPad triage queue in batches: new inbox, Blocking gaps,
  Soft gaps, blocked/watch holds, ranked/pursuing stages, or regional cluster
  passes. Load deals via list_deals, refresh stale market trends for the
  cluster, call research for data gaps, pattern-pass or underwrite, update
  statuses, and brief. Use when the user asks to triage, clear the queue, new
  inbox, blocking gaps, soft gaps, needs research, needs verification, blocked,
  regional pass, or batch-pass a market/builder cluster.
---

# RealtyPad — triage

Call `get_agent_manual(workflow="triage")` before running this workflow. Follow that Markdown.

## Hard rules

- Prefer evidence over Economics alone. Bare `score` = Economics; prefer `score_summary` + `data_gaps`.
- Any `data_gaps` entry with `blocking: true` means do not advance to score/UW. Prefer `has_blocking_gap=true` / `has_gap=<code>`.
- Fill gaps with the `research` skill before judging keepers. Hard external gaps → `blocked`.
- Regional/cluster: refresh once via `trends` (`ensure_deal_trends` per deal).
- Individual verdicts: `underwrite` (prefer `advance_deal` when score-ready). Pattern-pass only when the failure mode is documented.
- Status / Status note via **`update_deal_status`** (single) or status+rationale-only **`update_deals`** — never mix money fields in the same call.
- Read comments + open `action_needed` handoffs before flipping a previously researched deal.

## Queue modes

Match UI `/triage` (labels: New, **Blocking gaps**, **Soft gaps**, …):

- **new** (default inbox): `list_deals(status="new")` — start → `researching`, or pass / block.
- **Blocking gaps** (mode id `needs_research`): `list_deals(status="new,researching", has_blocking_gap=true)` — photos / rent / ADR / ARV / unscored. Or `has_gap="photos"`.
- **Soft gaps** (mode id `needs_verification`): `list_deals(needs_input=true)` or per-code `has_gap="tax_estimated"` — estimated tax/insurance, rent capped, `repairs_unknown`.
- **blocked / watch / ranked / pursuing**: `list_deals(status=…)`.
- **cluster**: same market/builder/pattern; refresh trends once, then batch.

## Status quick guide

| Status | When |
|--------|------|
| `new` | Just ingested; not started |
| `researching` | Actively working gaps / UW |
| `blocked` | Hard stop until an external fact clears |
| `watch` | Hold — no buyers yet (economics clear; empty book / parked — **not** more research) |
| `ranked` | Economics clear **and** ≥1 buyer match |
| `pursuing` | Human escalate from ranked |
| `passed` | Below bar, structural blocker, or no buyer fit |

## Aggressiveness

`ltr`/`str` moderate (prefer hold projections; fallback ~$100/mo). `fix_flip`/`wholesale` aggressive (asymmetric only). `strict` raises floors. Empty buyer book → `watch`; hard external gaps → `blocked`.

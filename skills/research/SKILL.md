---
name: research
description: >-
  Fill verified RealtyPad deal inputs from public sources (tax, HOA,
  insurance, rent/ADR/ARV, listing copy, lat/lng, photos) without ranking or
  scenario judgment; refresh stale market trends when needed. Use when
  researching a deal, clearing blocking/soft data_gaps, enriching before
  underwrite/scenarios, or when triage/underwrite needs evidence gathered first.
---

# RealtyPad — research

Call `get_agent_manual(workflow="research")` before running this workflow. Follow that Markdown.

## Hard rules

- Evidence only. Do not rank, score-judge, or set status / Status note here.
- Never invent ADR/ARV/rent/tax. Read `data_gaps` first — any `blocking: true` must clear before UW.
- Trends: prefer **`ensure_deal_trends(deal_id=…)`** (pass `neighborhood` when known). Do not LLM-parse catalog CSVs into tenant upsert.
- Money / listing copy via `update_deal` / money-only `update_deals`. Property structure via `update_deal_property`. **Never** `update_deal_status` from research.
- Comments for findings; `add_observation` only for `kind=action_needed` handoffs (not a research diary).
- CapEx → `cost-estimate` skill (repair items). Recurring OpEx → `add_deal_opex_items`. Never bump `maintenance_pct` as a proxy.
- Photos: attach full gallery when `has_gap="photos"` / blocking photos — do not only leave a missing-gallery note.

## Typical tools

`get_deal`, `update_deal`, `update_deal_property`, `ensure_deal_trends`, `add_comment`, `add_observation` (`action_needed` only), `list_deal_comps`, `find_deal_comps`, `add_deal_comps`, `list_deal_appraisal_tax`, `add_deal_appraisal_tax`, `add_deal_appraisal_tax_history`, `list_deal_attachments`, `add_deal_attachment_urls`, `add_deal_opex_items`.

## Checklist

```
Research Progress:
- [ ] Load deal + comments / action_needed / data_gaps
- [ ] ensure_deal_trends if markets stale/missing
- [ ] Fill tax / HOA / insurance / rent or ADR / ARV from public sources
- [ ] CapEx via cost-estimate; OpEx lines when STR/LTR recurring costs known
- [ ] Gap-fill photos, lat/lng, listing copy
- [ ] Persist comps and tax history (money-only patches)
- [ ] Stop. Hand off to scenarios / underwrite / triage
```

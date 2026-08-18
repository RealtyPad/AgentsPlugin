---
name: research
description: >-
  Fill verified RealtyPad deal inputs from public sources (tax, HOA,
  insurance, rent/ADR/ARV, listing copy, lat/lng, photos) without ranking or
  scenario judgment; refresh stale market trends when needed. Use when
  researching a deal, clearing needs-input data gaps, enriching before
  underwrite/scenarios, or when triage/underwrite needs evidence gathered first.
---

# RealtyPad — research

Call `get_agent_manual(workflow="research")` before running this workflow. Follow that Markdown.

## Hard rules

- Evidence only. Do not rank, score-judge, or change triage status here.
- Never invent ADR/ARV/rent/tax. Skip with `missing_adr` / `missing_arv` / etc. when the source is absent.
- Refresh stale market trends via the `trends` skill (`refresh_catalog_trends`) before using comps or hold growth.
- Persist findings: observations, comments, comps, appraisal/tax history, attachment gap-fill.

## Typical tools

`get_deal`, `update_deal`, `update_deal_property`, `add_observation`, `add_comment`, `list_deal_comps`, `find_deal_comps`, `add_deal_comps`, `list_deal_appraisal_tax`, `add_deal_appraisal_tax`, `add_deal_appraisal_tax_history`, `list_deal_attachments`, `add_deal_attachment_urls`.

## Checklist

```
Research Progress:
- [ ] Load deal + existing comments/observations
- [ ] Refresh stale linked-geo trends if needed
- [ ] Fill tax / HOA / insurance / rent or ADR / ARV from public sources
- [ ] Gap-fill photos, lat/lng, listing copy
- [ ] Persist comps and tax history
- [ ] Stop. Hand off to scenarios / underwrite / triage
```

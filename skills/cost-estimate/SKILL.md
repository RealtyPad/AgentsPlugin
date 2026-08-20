---
name: cost-estimate
description: >-
  Tighten RealtyPad deal CapEx with ZIP-local Home Depot materials (Apify) plus
  public labor/project cost ranges from web search, then persist structured
  repair items. Use when setting or revising CapEx for fix_flip / wholesale /
  needs_rehab deals. Aliases: repairs, repair-estimate.
---

# RealtyPad — cost / repair estimate

Call `get_agent_manual(workflow="cost-estimate")` before running this workflow. Follow that Markdown. Aliases: `repairs`, `repair-estimate`.

## Hard rules

- Evidence only. Never invent line items. Do not set `ranked` / `passed`.
- Persist structured CapEx via `add_deal_repair_item(s)` — not markdown BOM tables as source of truth. Writes roll up to `repair_estimate` and set `needs_rehab` when items exist.
- Materials via allowlisted Home Depot Apify Actors; labor via client web search (Angi / Homewyse / Fixr). Cap Apify spend with `maxTotalChargeUsd`.
- Prefer low confidence until a walkthrough or contractor bid exists.

## Typical tools

`get_deal`, `list_deal_repair_items`, `add_deal_repair_item`, `add_deal_repair_items`, `update_deal_repair_item`, `delete_deal_repair_item`, `add_deal_repair_lines`, `add_observation`, Home Depot `apify_*` tools.

## Checklist

```
Cost estimate:
- [ ] Load deal — condition notes, photos, existing repair_items / repair_estimate
- [ ] Scope line items from evidence only
- [ ] Materials: Home Depot Apify near deal ZIP
- [ ] Labor/project ranges: web search Angi/Homewyse/Fixr
- [ ] Persist via add_deal_repair_items (replace_all if rewriting)
- [ ] Optional estimate observation; confirm rollup with list_deal_repair_items
```

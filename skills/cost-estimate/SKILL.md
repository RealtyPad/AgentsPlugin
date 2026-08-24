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
- This skill owns **Acquisition CapEx** (`repair_items` → `repair_estimate` → `cash_in`). Recurring OpEx is separate (`opex_items`). **Never** bump `maintenance_pct` / `insurance_annual` as a CapEx proxy.
- Persist structured CapEx via `add_deal_repair_item(s)` — not markdown BOM tables as source of truth. When items exist they own the scalar (patches ignored); sets `needs_rehab`.
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

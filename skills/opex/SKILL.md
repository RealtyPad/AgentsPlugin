---
name: opex
description: >-
  Itemize RealtyPad recurring operating expenses (Monthly costs / OpEx) for
  LTR and STR deals: cleaning, utilities, supplies, property management,
  reserves, per-turnover costs. Persists opex items that roll into monthly
  cashflow. Use when the user asks for monthly costs, operating expenses,
  OpEx, STR cleaning/turnover costs, or when research/underwrite needs
  recurring costs itemized.
---

# RealtyPad — monthly costs (OpEx)

Evidence-only recurring operating expenses. **Never invent line items.** Seed from workspace templates when useful; price local services via search when needed. Do not set `ranked` / `passed`.

## Mental model

| Bucket | Persist how | Feeds | Never | UI label |
|--------|-------------|-------|-------|----------|
| **Recurring OpEx** | `add_deal_opex_item(s)` | `monthly_opex_total` → monthly CF | Day-0 rehab BOM; invent rates | **Monthly costs** |
| **Acquisition CapEx** | `add_deal_repair_item(s)` via the **`cost-estimate`** skill | `repair_estimate` → `cash_in` | Bump into OpEx / `maintenance_pct` | **Fix-up costs** |

CapEx playbook stays in **cost-estimate**. This workflow owns **Monthly costs** only.

## Hard rules

- **Never invent** amounts — prefer null / skip / soft gap over fiction.
- **Never** bump `maintenance_pct` or `insurance_annual` as a blended OpEx proxy — itemize lines instead.
- **Never** put day-0 rehab (paint, roof, kitchen) into OpEx — that is CapEx / cost-estimate.
- **Never** duplicate the hard-coded STR **20% platform fee** already in `str_v1` as an opex line.
- Scope by trigger `income_strategies`, else deal `strategy_intent`, else active `income_strategy`. Owner-occ (`deal_purpose=primary`) has no Expenses OpEx path — skip.
- Prefer **`list_deal_opex_items`** before writing; avoid duplicate name+category+scope.
- Optional starters: `get_uw_cost_defaults().opex_templates` + `default_avg_stay_nights` for per-turnover math.
- Writes roll into `deals.monthly_opex_total` and rescore CF.

## Strategy scope (LTR vs STR)

| Focus | Typical lines | Notes |
|-------|---------------|-------|
| **LTR** | utilities (if landlord-paid), landscaping, PM (`pct_of_rent`), reserves | Vacancy/maintenance % stay on deal scalars unless itemized |
| **STR** | cleaning, utilities, supplies, PM, `capex_reserve`, laundry, pool — often `per_turnover` or monthly | Do **not** add Airbnb/VRBO platform fee |
| **both** | shared lines with `income_strategy_scope=both` | Prefer scope `ltr` / `str` when amounts differ |

## Checklist

```
Monthly costs (OpEx):
- [ ] 1. get_deal + list_deal_opex_items — existing lines + strategy focus
- [ ] 2. get_uw_cost_defaults — opex_templates, default_avg_stay_nights
- [ ] 3. Evidence search (Firecrawl / local rates) only when a line needs a quote
- [ ] 4. Persist via add_deal_opex_item(s) (replace_all only when rewriting stubs)
- [ ] 5. Confirm monthly_opex_total / rescore; note gaps in research comment if thin
```

## MCP tools

| Tool | Use |
|------|-----|
| `list_deal_opex_items` | Read current lines + rollup |
| `add_deal_opex_item` | One line |
| `add_deal_opex_items` | Bulk (max 100); `replace_all=true` for full rewrite |
| `update_deal_opex_item` / `delete_deal_opex_item` | Edit / remove |
| `get_uw_cost_defaults` | Workspace templates + default avg stay nights |

**Fields:** `name` (required); `frequency` = `daily` \| `weekly` \| `biweekly` \| `monthly` \| `annual` \| `per_turnover` \| `pct_of_rent`; `amount` (required unless `pct_of_rent`); `basis_pct` when `pct_of_rent`; `turns_per_month` for turnover math; `category` (`cleaning`, `utilities`, `supplies`, `pm`, `capex_reserve`, …); `income_strategy_scope` = `ltr` \| `str` \| `both`; `source_url` / `notes` when known.

## Hosted tab research

Expenses → **Monthly costs** → **Research monthly costs** enqueues a refresh automation (`kind=opex`); the worker follows this playbook and saves with `add_deal_opex_item(s)`.

## See also

- CapEx / Fix-up costs: the **`cost-estimate`** skill
- Underwrite OpEx step: the **`underwrite`** skill §3c
- Research Expenses model: the **`research`** skill

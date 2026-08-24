---
name: underwrite
description: >-
  Full underwrite of a RealtyPad cashflow or wholesale deal: call research
  for verified inputs if needed, prefer advance_deal when score-ready, ask
  which linked geo feeds UW worksheet / hold growth when multi-geo trends
  exist, run scenarios for persisted scenario runs, set triage status, leave
  an agent comment. Use when the user asks to underwrite, UW, fully underwrite,
  dig into a deal, advance a deal, or clear a new / researching / blocked lead.
---

# RealtyPad — underwrite

Call `get_agent_manual(workflow="underwrite")` before running this workflow. Follow that Markdown.

## Hard rules

- Never invent numbers. If research is incomplete (blocking `data_gaps`), run `research` first.
- Prefer **`advance_deal(dry_run=true)`** then `dry_run=false` when score-ready. Stops without write on blocking gaps, `deal_purpose=primary`, auction (`needs_distress_review`), or multi-geo without `primary_geography_level`.
- When multi-geo trends exist, ask which linked geo feeds the UW worksheet / hold growth. Prefer `ensure_deal_trends` for freshness.
- CapEx = repair items → `cash_in`. OpEx = `opex_items` → monthly CF. **Never** bump `maintenance_pct` / `insurance_annual` as a CapEx/OpEx proxy.
- Buyer-fit gate: ≥1 match → `ranked`; scanned none → `passed`; empty book → **`watch`** (Hold — no buyers yet). Hard external gaps → **`blocked`**.
- Distress: stay `researching`/`blocked` until tax/title/repairs are credible, then apply the gate.
- Status + short Status note (`rationale`) via **`update_deal_status`**. Full brief via `add_comment`. Money-only via `update_deal` — never mix status into money patches.

## Aggressiveness

| Strategy | Mode | Clears economic bar when |
|----------|------|--------------------------|
| `ltr` / `str` | moderate | Prefer hold projections; fallback CF ≳ ~$100/mo |
| `fix_flip` / `wholesale` | aggressive | Asymmetric only |

`strict` raises floors (~$200/mo / flip ≥$50k / wholesale ≥$40k).

## Checklist

```
Underwrite Progress:
- [ ] get_deal + comments / action_needed + data_gaps
- [ ] Trends geo choice if multiple linked geos (ensure_deal_trends)
- [ ] Prefer advance_deal(dry_run=true) → review → dry_run=false
- [ ] Else: scenarios persist + select → match_deal_buyers
- [ ] Status note (update_deal_status) + UW comment (unless advance_deal wrote)
```

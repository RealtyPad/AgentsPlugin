---
name: underwrite
description: >-
  Full underwrite of a RealtyPad cashflow or wholesale deal: call research
  for verified inputs if needed, ask which linked geo feeds UW worksheet /
  hold growth when multi-geo trends exist, run scenarios for persisted
  scenario runs, set triage status, leave an agent comment. Use when the
  user asks to underwrite, UW, fully underwrite, dig into a deal, or clear
  a new / researching / blocked lead.
---

# RealtyPad — underwrite

Call `get_agent_manual(workflow="underwrite")` before running this workflow. Follow that Markdown.

## Hard rules

- Never invent numbers. If research is incomplete, run `research` first.
- When multi-geo trends exist, ask which linked geo feeds the UW worksheet / hold growth.
- Persist scenarios (`scenarios` skill) before writing a verdict.
- Buyer-fit gate: `match_deal_buyers` after economics clear. ≥1 match → `ranked`; scanned none → `passed`; empty book → **`watch`**. Hard external gaps → **`blocked`**.
- Distress: stay `researching`/`blocked` until tax/title/repairs are credible, then apply the gate.
- Pass/Block require a short rationale note. Write the full UW comment too. Status via `update_deal` / `update_deals`. Rescore with `rescore_deal` after input changes.

## Aggressiveness

| Strategy | Mode | Clears economic bar when |
|----------|------|--------------------------|
| `ltr` / `str` | moderate | Prefer hold projections; fallback CF ≳ ~$100/mo |
| `fix_flip` / `wholesale` | aggressive | Asymmetric only |

`strict` raises floors (~$200/mo / flip ≥$50k / wholesale ≥$40k).

## Checklist

```
Underwrite Progress:
- [ ] get_deal + comments + research gaps
- [ ] Trends geo choice if multiple linked geos
- [ ] Scenarios persist + select
- [ ] match_deal_buyers
- [ ] Status + UW comment
```

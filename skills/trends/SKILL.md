---
name: trends
description: >-
  Check and refresh RealtyPad market trends for a deal or cluster. Prefer
  ensure_deal_trends as the entrypoint (geo chain + live APIs + missing CSV).
  Power tools refresh_catalog_trends / refresh_airroi_trends only when surgical.
  Tenant upsert is for custom/override sources only. Use when the user asks to
  update trends, refresh ZHVI/ZORI/AirROI, or when research/triage needs fresh
  market series.
---

# RealtyPad — trends

Call `get_agent_manual(workflow="trends")` before running this workflow. Follow that Markdown.

## Hard rules

- Never invent numbers. Percent fields are **fractions** (YoY −3.2% → `-0.032`).
- **Preferred:** `ensure_deal_trends(deal_id=…, neighborhood=… when known)`. Gap-fills live APIs when stale/missing; CronJob CSV when **missing** only (not merely-stale unless `force_csv=true`).
- Do **not** LLM-parse catalog CSVs into `upsert_market_snapshots_bulk`. Tenant upsert = custom/Redfin only.
- Power tools: `refresh_catalog_trends` / `refresh_airroi_trends` only for surgical force or single-source gaps. Prefer reading merged snapshots after ensure.
- Target up to ~10y (~120 mo) where the source allows (AirROI max 60). Do not change deal status from a trends pass.

## Typical tools

`ensure_deal_trends`, `ensure_geo_markets`, `list_markets`, `list_market_snapshots`, `get_latest_market_snapshot`, `list_catalog_snapshots`, `get_latest_catalog_snapshot`, `get_market_projections`, `refresh_catalog_trends`, `refresh_airroi_trends`, `upsert_market_snapshot`, `upsert_market_snapshots_bulk`.

## Checklist

```
Trends Progress:
- [ ] ensure_deal_trends (pass neighborhood when known)
- [ ] Read markets[].latest_by_source
- [ ] Power tools only if surgical force / single-source gap
- [ ] Brief geo × series matrix
```

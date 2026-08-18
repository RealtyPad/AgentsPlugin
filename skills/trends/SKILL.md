---
name: trends
description: >-
  Check and refresh RealtyPad market trends (home-value, LTR rent, STR, and
  supply/demand) for a market or cluster. Read merged snapshots first; use
  refresh_catalog_trends for missing/stale points or API sources. Deal-scoped
  STR uses refresh_airroi_trends. Tenant upsert is for custom/override sources
  only. Use when the user asks to update trends, refresh ZHVI/ZORI/AirROI, or
  when research/triage needs fresh market series.
---

# RealtyPad — trends

Call `get_agent_manual(workflow="trends")` before running this workflow. Follow that Markdown.

## Hard rules

- Never invent numbers. Percent fields are **fractions** (YoY −3.2% → `-0.032`).
- Read merged snapshots first (`list_market_snapshots`, `get_latest_market_snapshot`, `get_latest_catalog_snapshot`, `get_market_projections`).
- On “update trends,” cover **all linked geos × every published catalog series** via `refresh_catalog_trends` (ZHVI/ZORI/listings/Census/FRED; STR `airroi` when relevant).
- Deal-scoped STR: `refresh_airroi_trends`. Tenant `upsert_market_snapshots_bulk` is for custom/Redfin overrides only.
- Target up to ~10y (~120 mo) where the source allows (AirROI max 60).

## Typical tools

`ensure_geo_markets`, `list_markets`, `list_market_snapshots`, `get_latest_market_snapshot`, `list_catalog_snapshots`, `get_latest_catalog_snapshot`, `get_market_projections`, `refresh_catalog_trends`, `refresh_deal_trends`, `refresh_airroi_trends`, `upsert_market_snapshot`, `upsert_market_snapshots_bulk`.

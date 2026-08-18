---
name: ingest
description: >-
  Pull market listings into RealtyPad from any available connector (Apify,
  Firecrawl, user scraping MCP, or MLS MCP), normalize, upsert via
  add_manual_leads (batches) or add_manual_lead (single), attach photos, capture
  listing_description + lat/lng. Use when the user asks for a sweep, scrape,
  ingest, HUD/Zillow/Auction/Craigslist pull, MLS import, market lead pull, or
  listing enrich/backfill.
---

# RealtyPad — ingest

Call `get_agent_manual(workflow="ingest")` before running this workflow. Follow that Markdown. The notes below are a short checklist, not a second source of truth.

## Hard rules

- Never invent numbers. Always pass `source_url` and/or `listing_urls` for dedupe.
- Search cards are not ingest. Open the **listing detail page** (or detail Actor) before calling the sweep done.
- Pass the **full gallery** as `image_urls` on create (first = primary), plus `latitude`/`longitude` and `listing_description`.
- Stop after ingest + detail enrich. Research, triage, and underwrite are separate skills.
- Do not scrape ToS-hostile sites without explicit user approval. Cap spend on paid connectors.

## Connector order

Prefer the first available fit: user MLS MCP → user scraping MCP → RealtyPad `apify_*` tools → Firecrawl → pasted URLs / sheets.

## Checklist

```
Ingest Progress:
- [ ] Confirm geography, income_strategy, price ceiling, lead_source, connector
- [ ] Pull search/list results
- [ ] Detail-pull each keep
- [ ] Normalize → add_manual_leads (or add_manual_lead)
- [ ] Gap-fill photos only if the gallery was thin
- [ ] Confirm description + lat/lng + gallery + beds/baths/sqft/year/lot + amenities
- [ ] Brief counts; do not auto-research/UW unless asked
```

Property fields go on `update_deal_property`. Verbatim remarks go on `update_deal` (`listing_description`) when not set at create.

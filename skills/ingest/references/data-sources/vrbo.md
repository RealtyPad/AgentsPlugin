# VRBO (listing connector)

## Role

STR **geo comps** — city/ZIP search returns calendar ADR / occupancy / RevPAR. **Not** a for-sale listing ingest path (no sale `ListingDetailAdapter`).

## Lead source

`vrbo` when tagging comps / channel; not for acquisition capture of homes for sale.

## Tools

| Job | Actor id | MCP tool |
|-----|----------|----------|
| VRBO geo | `memo23/vrbo-scraper` | `memo23--vrbo-scraper` |

## Input recipe

- City/ZIP/geo `startUrls` (confirm schema via `fetch-actor-details`).
- Optional `scrapeAvailability` for calendar KPIs.
- Cap `maxTotalChargeUsd` — geo runs can be spendy.

## Gotchas

- Comps / research only — do not create sale deals from VRBO rows as if they were MLS inventory.
- **Booking.com is not allowlisted**; use VRBO + AirROI / [AirDNA](airdna.md) instead.
- Deal needs context (strategy STR in focus) before patching ADR/occ from comps.

## Persist hints

- `add_deal_comps(kind=str, source=vrbo)` with ADR nightly / occupancy evidence; include `image_url` when present.
- Patch deal `adr_nightly` / `occupancy_pct` only after evidence — not from comps medians alone.
- Market-level STR series: the **`trends`** skill.

## Related

- Research: the **`research`** skill
- Catalog: the data-source catalog ([data-sources.md](../data-sources.md))

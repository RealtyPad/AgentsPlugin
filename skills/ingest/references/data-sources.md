# RealtyPad — data sources (listing connectors)

Allowlisted **listing connectors** mounted on RealtyPad MCP (no separate Apify account). Prefer this catalog over inventing Store Actors.

**Load:** the data-source catalog ([data-sources.md](data-sources.md)) (this overview) → ``data-sources/<portal>.md`` for usage and gotchas.

## Hard rules

- **Allowlist only.** Dedicated tools are `<user>--<actor>` plus `fetch-actor-details` / run helpers (`get-actor-run`, `get-dataset-items`, …). Generic `call-actor` / `search-actors` and unrestricted web-browser Actors are **not** available.
- Cap spend with `maxRecords` and/or `maxTotalChargeUsd` before every run.
- Prefer Firecrawl / MLS MCP for a single known URL when cheaper than a paid Actor.
- **Never** use `apify/rag-web-browser` or other non-allowlisted Store Actors.
- **Booking.com is not allowlisted** (hotel-heavy; weak vacation-home filter). Use [VRBO](data-sources/vrbo.md) + AirROI / [AirDNA](data-sources/airdna.md) for STR comps.

## Run pattern

1. `fetch-actor-details` on an allowlisted Actor (read input schema; keep `output.outputSchema` false unless needed)
2. Call the dedicated tool (e.g. `martc03--hud-foreclosures`)
3. `get-actor-run` to poll if needed
4. `get-dataset-items` (paginate if large)
5. Normalize → ingest / comps / CapEx lines — see workflow guides

## Catalog

| Source | Role | Lead source | Detail page |
|--------|------|-------------|-------------|
| Zillow | Sale search → detail; rent/sale comps | `zillow` | [zillow.md](data-sources/zillow.md) |
| Redfin | Sale search | `redfin` | [redfin.md](data-sources/redfin.md) |
| Realtor.com | Sale search (address URL) | `realtor_com` | [realtor-com.md](data-sources/realtor-com.md) |
| Trulia | Sale search → detail | `trulia` | [trulia.md](data-sources/trulia.md) |
| HUD / REO | Foreclosure / REO lists | `hud` | [hud.md](data-sources/hud.md) |
| Auction.com | Auction inventory / bids | `auction_com` | [auction-com.md](data-sources/auction-com.md) |
| Craigslist | FSBO / CL | `craigslist` | [craigslist.md](data-sources/craigslist.md) |
| Open-data distress | Socrata probate / foreclosure / tax lien | (varies) | [probate-foreclosure.md](data-sources/probate-foreclosure.md) |
| AirDNA | Known Airbnb URL STR enrich | — | [airdna.md](data-sources/airdna.md) |
| VRBO | STR geo comps (not sale ingest) | `vrbo` | [vrbo.md](data-sources/vrbo.md) |
| Home Depot | CapEx materials | — | [home-depot.md](data-sources/home-depot.md) |

`realtor` (human agent / off-MLS) ≠ `realtor_com` (Realtor.com portal).

## Related (not Apify connectors)

| Source | Use | Where |
|--------|-----|--------|
| AirROI MCP | STR listing comps + catalog STR series | `airroi_listings_comparables`; the **`trends`** skill |
| Catalog trends | ZHVI/ZORI/listings/FRED/BPS/… | the **`trends`** skill |
| Firecrawl / MLS BYO | Single URL or board feed | Client MCP; ingest/research preference order |
| Manual / sheet | Pasted URLs | `add_manual_lead(s)` |

## Related workflows

- Ingest normalize / galleries: the **`ingest`** skill
- Comps + tax gap-fill: the **`research`** skill
- Auction / REO offer stack: the **`distress`** skill
- CapEx BOM: the **`cost-estimate`** skill

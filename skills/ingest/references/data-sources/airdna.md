# AirDNA (listing connector)

## Role

Enrich **known** Airbnb listing URLs with AirDNA metrics (ADR / occupancy / revenue when unlocked). **Not** a market finder and **not** sale ingest.

## Lead source

Optional `source=airdna` on comps; not a sale `lead_source_code`.

## Tools

| Job | Actor id | MCP tool |
|-----|----------|----------|
| Airbnb / AirDNA | `corent1robert/airdna-scraper` | `corent1robert--airdna-scraper` |

## Input recipe

1. `fetch-actor-details` → confirm schema.
2. Call with:
   - `airbnbListingUrls`: `https://www.airbnb.com/rooms/...` (one or more)
   - `currency`: `usd` for US deals (Actor default is often `eur`)
   - `marketId`: AirDNA market id when known — without it, revenue / occupancy / revenue_potential are often redacted
3. Cap `maxTotalChargeUsd`.
4. `get-dataset-items` → map metrics.

## Gotchas

- Prefer platform-configured Actor secrets (`AIRDNA_*`). Do **not** put email/password/bearer in tool args unless a workspace admin explicitly asks.
- If the run fails on auth/rental/permissions, stop and tell the user — do not switch to another Actor.
- Prefer AirROI listings, [VRBO](vrbo.md) geo, or web STR comps as market finders. **Booking.com is not allowlisted.**
- Never invent ADR/occ.

## Persist hints

- Deal: `update_deal` `adr_nightly`, `occupancy_pct` only after evidence.
- Comps: `add_deal_comp(kind=str)` with `source=airdna` + listing URL; persist `image_url` when present.
- Market-level STR series still go through the **`trends`** skill / `refresh_airroi_trends` (catalog ≠ this Actor).

## Related

- Research comps: the **`research`** skill
- Catalog: the data-source catalog ([data-sources.md](../data-sources.md))

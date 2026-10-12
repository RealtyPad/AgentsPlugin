# HUD / REO (listing connector)

## Role

HUD / REO foreclosure lists — cheap public distress inventory. Often paired with Auction.com when sweeping a metro.

## Lead source

`hud`

## Tools

| Job | Actor id | MCP tool |
|-----|----------|----------|
| HUD / REO | `martc03/hud-foreclosures` | `martc03--hud-foreclosures` |

## Input recipe

- Filter by state / ZIP / price when the schema supports it.
- Cap `maxRecords` / `maxTotalChargeUsd`.
- Search/list hits are cards — detail-enrich (Firecrawl or listing page) before calling ingest done.

## Gotchas

- Prefer public/gov sources first; respect ToS.
- For assign-to-flipper / wholesale playbooks see distress workflow.

## Persist hints

`lead_source_code=hud`, `ask_price` from list when present, `auction_date` if shown, then full gallery + write-up from detail. Strategy often `wholesale` when running distress sweeps.

## Related

- Distress: the **`distress`** skill
- Ingest: the **`ingest`** skill
- Catalog: the data-source catalog ([data-sources.md](../data-sources.md))

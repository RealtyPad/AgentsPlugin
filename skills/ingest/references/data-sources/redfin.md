# Redfin (listing connector)

## Role

Sale **search** inventory. Detail-enrich keeps with Firecrawl / portal detail or another allowlisted detail path before treating a card as ingest.

## Lead source

`redfin`

## Tools

| Job | Actor id | MCP tool |
|-----|----------|----------|
| Search | `tri_angle/redfin-search` | `tri_angle--redfin-search` |

## Input recipe

- Cap records and `maxTotalChargeUsd`.
- Read schema via `fetch-actor-details` before first run (location / filter fields vary by Actor version).

## Gotchas

- Search rows are cards — not enough for underwrite. Persist full gallery + write-up from a detail pull.
- Photo arrays on Redfin/MLS-style payloads map to `image_urls` the same way as other portals.

## Persist hints

Normalize into `add_manual_lead(s)` with `lead_source_code=redfin`, `source_url`, beds/baths/sqft when present, then detail-enrich.

## Related

- Ingest: the **`ingest`** skill
- Catalog: the data-source catalog ([data-sources.md](../data-sources.md))

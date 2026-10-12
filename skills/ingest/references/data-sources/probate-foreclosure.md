# Open-data distress (listing connector)

## Role

Socrata-backed probate / foreclosure / tax-lien style leads (open data), not a consumer portal card.

## Lead source

Use the closest seeded code from `list_lead_sources` (often distress-related); note the dataset in `notes`. Prefer `hud` / `auction_com` only when the row is truly from those portals.

## Tools

| Job | Actor id | MCP tool |
|-----|----------|----------|
| Open-data distress | `jungle_synthesizer/probate-foreclosure-leads-scraper` | `jungle_synthesizer--probate-foreclosure-leads-scraper` |

## Input recipe

- Cap records and `maxTotalChargeUsd`.
- Confirm geography / dataset filters via `fetch-actor-details`.

## Gotchas

- Open-data rows can be stale or incomplete addresses — verify before underwrite.
- Pair with Auction.com / HUD when building a distress book; see distress workflow for offer stack.

## Persist hints

Normalize carefully; never invent liens or tax. Store connector run id in `notes`.

## Related

- Distress: the **`distress`** skill
- Ingest: the **`ingest`** skill
- Catalog: the data-source catalog ([data-sources.md](../data-sources.md))

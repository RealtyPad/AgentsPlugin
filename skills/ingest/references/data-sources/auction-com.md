# Auction.com (listing connector)

## Role

Auction inventory and live/starting bids. Primary channel for assign-to-flipper / wholesale distress sweeps.

## Lead source

`auction_com`

## Tools

| Job | Actor id | MCP tool |
|-----|----------|----------|
| Auction.com | `abotapi/auction-com-scraper` | `abotapi--auction-com-scraper` |

## Input recipe

- Typical: `mode=search`, `states`, `city`, `maxListings`, `fetchDetails=true` (confirm with `fetch-actor-details`).
- Cap `maxListings` / `maxTotalChargeUsd`.
- Re-fetch detail before bidding — bids move.

## Gotchas

- `ask_price` = **starting bid or live current bid** — say which in notes.
- Reserve not met + large increments means opening bid is not a clearing price.
- Cotality / AVM on the portal = **estimate** (sale comp `status=estimate`), not closed comps.
- Annual tax ≠ delinquent / lien payoff — leave null until verified.
- Prefer this Actor over stale alternate Auction.com Store ids.

## Persist hints

| Field | Value |
|-------|--------|
| `income_strategy` | often `wholesale` |
| `ask_price` | startingBid or live current bid |
| `auction_date` | Sale / auction day when known |
| `liens_owed` | Only when verified — never invent |
| Gallery | Auction.com / imgix photo set → `image_urls` |

## Related

- Distress offer stack: the **`distress`** skill
- Ingest: the **`ingest`** skill
- Catalog: the data-source catalog ([data-sources.md](../data-sources.md))

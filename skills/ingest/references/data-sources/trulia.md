# Trulia (listing connector)

## Role

Sale **search** (thin cards) then **detail** enrich (description, gallery, HOA). Pair like Zillow: search → detail on every keep.

## Lead source

`trulia`

## Tools

| Job | Actor id | MCP tool |
|-----|----------|----------|
| Search | `igolaizola/trulia-scraper-ppe` | `igolaizola--trulia-scraper-ppe` |
| Detail | `memo23/trulia-scraper` | `memo23--trulia-scraper` |

## Input recipe

### Search (`igolaizola/trulia-scraper-ppe`)

- `location` = full street address or city/ZIP; `operation=buy`.
- Small `maxItems`; pick street match.
- Returns search cards (URL, price, beds, thin media) — **not** gallery/description source.

### Detail (`memo23/trulia-scraper`)

- Pass Trulia `/home/…` URLs from the search row as `startUrls`.
- Cap `maxTotalChargeUsd`.

## Gotchas

- Do not treat search as the gallery/description source.
- A known `/home/…` detail URL can go straight to memo23 without a search pass.

## Persist hints

| From detail | Into |
|-------------|------|
| `rawPrice` | `ask_price` |
| beds/baths, `floorSpace` | property fields |
| `photos` / `totalPhotoCount` | `image_urls` |
| `description` | `listing_description` |
| coords, `hoaFee` | property / `hoa_monthly` |
| Agent from `lastSoldDetails` / provider | `link_deal_contact` when present |

## Related

- Ingest: the **`ingest`** skill
- Catalog: the data-source catalog ([data-sources.md](../data-sources.md))

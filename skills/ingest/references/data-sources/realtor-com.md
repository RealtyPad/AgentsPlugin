# Realtor.com (listing connector)

## Role

Sale **search** by address-shaped URL (or careful location search). Prefer for portal sweeps when Zillow is thin or the user provides a Realtor.com link.

## Lead source

`realtor_com` (portal). Do **not** use `realtor` — that is the human-agent / off-MLS channel.

## Tools

| Job | Actor id | MCP tool |
|-----|----------|----------|
| Search | `memo23/realtor-search-cheerio` | `memo23--realtor-search-cheerio` |

## Input recipe

- Prefer address-shaped **search** URLs:  
  `https://www.realtor.com/realestateandhomes-search/{Street}-{City}-{ST}-{zip}` (spaces → hyphens).
- Cap `maxItems` (e.g. 5–10); prefer the row whose address matches the street.
- Or pass `searchLocations` carefully — free-text location often returns unrelated city listings.

## Gotchas

- Bare `/realestateandhomes-detail/…` **without** an `_M…` MLS-style id **fails** — do not invent detail URLs.
- Only use `/realestateandhomes-detail/…_M…` when you already have that suffix from a prior search hit.

## Persist hints

| From payload | Into |
|--------------|------|
| `list_price` | `ask_price` |
| beds/baths/sqft/year, `lot_sqft` | property fields (lot → acres ÷ 43560) |
| `href` | `source_url` |
| `photo_urls` | `image_urls` |
| `description_text` | `listing_description` |
| lat/lng, `hoa_fee`, `latest_tax_amount` | deal money / property when present |
| Agent fields | `link_deal_contact` `role=listing_agent` |

## Related

- Ingest: the **`ingest`** skill
- Catalog: the data-source catalog ([data-sources.md](../data-sources.md))

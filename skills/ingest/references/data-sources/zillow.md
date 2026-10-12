# Zillow (listing connector)

## Role

Sale **search** inventory and **detail** enrich (write-up, gallery, lat/lng, facts, tax history). Also rent/sale **comps** near a subject when research needs them.

**When not:** Do not treat search cards as ingest. Do not use search `fetchDetails` as the gallery/tax source.

## Lead source

`zillow`

## Tools

| Job | Actor id | MCP tool |
|-----|----------|----------|
| Search | `igolaizola/zillow-scraper-ppe` | `igolaizola--zillow-scraper-ppe` |
| Detail | `maxcopell/zillow-detail-scraper` | `maxcopell--zillow-detail-scraper` |

## Input recipe

### Search (`igolaizola/zillow-scraper-ppe`)

- **One ZIP per run:** `location` + `locationType: "zipcode"`. Do **not** batch `locations: ["78745", "78644"]` — that is a single blended radius (second ZIP often returns zero; adjacent ZIPs leak in).
- Cap `maxRecords` / `maxTotalChargeUsd`.
- `fetchDetails: true` is still a **card-plus-facts** pull. `media.allPropertyPhotos.highResolution` is often empty.

### Detail (`maxcopell/zillow-detail-scraper`) — supported path

- Pass **`addresses`**: free-text `"<street>, <city>, <ST> <zip>"` from the search row (or the listing). No URL slug required.
- Cap `maxTotalChargeUsd`.
- Same run is the photo + `listingTaxHistory` gap-fill.

## Gotchas

- These two allowlisted Actors **do not chain**.
- Do **not** feed `startUrls` with bare `https://www.zillow.com/homedetails/<zpid>_zpid/` from the search Actor’s `url` — those fail (“No Zillow details to scrape”).
- Do **not** set `searchResultsDatasetId` to an igolaizola dataset. That field is for a **different**, non-allowlisted ZIP search Actor.
- Do not treat Zestimate as rent or ARV (comment / `raw` estimate only).
- **Sold records (`RECENTLY_SOLD` / off-market):** `listingPrice.amount` is **0** and the `Sold` row in `listingPriceHistory` usually has **no price**. Never use either as the sale price. Use the last `Pending` / `Contingent` price before that `Sold` row; if there is none, the last `Listed` / price-change price (say so in the comp note). A record whose history is only a price-less public-record `Sold` row is **unusable** — skip it, don't add a $0 or guessed comp.
- `get-dataset-items` dot-notation projection (`fields=listingPriceHistory.event`) returns empty for nested arrays — fetch the raw record or project the whole `listingPriceHistory` field.

## Persist hints

| From detail | Into |
|-------------|------|
| `description` / `homeDescription` | `listing_description` |
| `media.allPropertyPhotos.highResolution` (etc.) | `image_urls` / `add_deal_attachment_urls` — skip Google map/satellite |
| lat/lng | create `latitude`/`longitude` or `update_deal_property` |
| beds/baths/sqft/year/lot | create / `update_deal_property` (lot sqft → acres ÷ 43560) |
| HOA if listed | `update_deal` `hoa_monthly` |
| `listingTaxHistory` / `_details.taxHistory` | `add_deal_appraisal_tax_history` |
| `priceHistory` / listing timeline events | `add_deal_listing_events` (feeds Time on market / True time on market); rent rows (`Listed for rent`, `postingIsRental`) → `listing_type=rental` |
| Agent attribution | `link_deal_contact` `role=listing_agent` |
| Rent/sale comps rows | `add_deal_comps` (`source=zillow`); always persist `image_url` when present |
| Sold detail record → sale comp | price from the last pending/contingent row before `Sold` (see Gotchas); `as_of` = sold date; `status=sold` |

For duplex/triplex/multi_family **rent/STR** comps, query **per unit beds**; for **sale**, `homeTypes=multifamily` and unpin combined size.

## Related

- Ingest: the **`ingest`** skill
- Research / comps: the **`research`** skill
- Catalog: the data-source catalog ([data-sources.md](../data-sources.md))

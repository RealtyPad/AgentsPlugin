---
name: import
description: >-
  Pull market listings into RealtyPad: search a portal with utility_listing_search
  (or read one listing with utility_listing_get), pick the keepers, then create
  deals with deal_import by source_id. Covers sources, filters, dedupe, photos,
  listing agent, and enrichment checks. Use when the user asks for a sweep,
  scrape, import, HUD/Zillow/Auction/Craigslist pull, market lead pull, listing
  enrich/backfill, or which listing sources are available.
---

# RealtyPad — import (search → pick → import)

v2 flow: **search with `utility_*` → pick → write by `source_id`.** The `utility_*` tools only read (they never create a deal). `deal_import` is the only write. You never pass listing rows through your own context, and you never name a scraper: a `source` is a portal name.

## Hard rules

- **Never invent numbers.** Null > fiction. Mark estimates.
- **Import by `source_id` only.** A `source_id` comes from `utility_listing_search` or `utility_listing_get` and expires (cached about a day). On `source_expired`, search again; nothing was written.
- Dedupe is street + unit + ZIP5 (listing URL is the fallback when the ZIP is missing). `deal_import` returns `deduped` per row. `on_duplicate=update` (default) merges the fresh listing onto the existing deal and keeps its capture source; `on_duplicate=skip` leaves it unchanged. Check for an existing deal first with `deal_list(address=…)` or `deal_list(listing_url=…)` when the user asked about specific addresses.
- **Trust but verify on re-import.** The import carries the portal's current ask and status. After a re-import, compare the stored `ask_price` with the portal price. When they differ, write the new value with `deal_update` and say so.
- Re-import of an **archived** deal updates money fields but does **not** unarchive. Use `deal_update(archived=false)` to restore.
- Do **not** pull from ToS-hostile sites without explicit user approval (the **`realtypad`** skill).
- Prefer public and government sources first. Use a connected MLS MCP when the user has one (it is not a RealtyPad source).
- **Stop after import + enrich check** when the user only asked for an import. Hosted `goal=import` (Quick analyze, `automation_run_create`) continues into research and must leave a go/no-go package, including qualitative growth drivers (the **`growth-drivers`** skill). Research, triage and underwrite are separate goals.
- **A search hit is a pointer.** A search card has a price, beds and one photo. Read the detail (`utility_listing_get`) for every keeper and check the enrich list below before calling the sweep done.

## Sources

`source` is required on `utility_listing_search`. Each source takes only its native filters; any other filter is an `unsupported_filter` error that lists what the source supports.

| `source` | Location input | Native filters | Use for |
|---|---|---|---|
| `zillow` | `zip_code` (one ZIP per call) | `min_price`, `max_price`, `min_beds`, `min_baths`, `min_sqft`, `max_sqft`, `property_types` | Default sale search |
| `redfin` | `zip_code` only | location only | Sale search |
| `realtor` | `zip_code`, or `city` + `state` | location only | Sale search (Realtor.com) |
| `trulia` | `zip_code`, or `city` + `state` | location only | Sale search |
| `hud` | `zip_code` or `state` | location only | HUD / REO foreclosures |
| `auction` | `zip_code`, or `city` + `state` | location only | Auction.com inventory |
| `craigslist` | `city` (a known region) | `min_price`, `max_price`, `min_beds` | FSBO |

Probate and open-data distress lists are not a source yet. Say so and offer web research (the **`distress`** skill). Short-term-rental comps and materials are not listings: use `utility_comp_search` (the **`research`** skill) and `utility_material_search` (the **`cost-estimate`** skill).

`realtor` (the portal) is a source name. The capture source on the deal, `lead_source_code`, is separate: see `lead_source_list`.

## Workflow

```
Import Progress:
- [ ] 1. Confirm geography, income_strategy, price ceiling, source, lead source
- [ ] 2. utility_listing_search(source, zip_code | city+state, filters, max_results)
- [ ] 3. Pick keepers from the items (ask, beds, baths, status); drop stale, pending and non-residential
- [ ] 4. utility_listing_get(source_id) for each keeper you doubt (full detail, photos, description)
- [ ] 5. deal_import(source_ids) in batches of up to 100
- [ ] 6. Enrich check (below); gap-fill with deal_update / deal_item_upsert
- [ ] 7. Brief counts and standouts (do not auto-research or underwrite unless asked)
```

- Cost: each search is one provider run, cached for a day (an empty result for an hour). Repeating the same query is a cache hit. Use `refresh=true` only when the user wants a re-run. Keep `max_results` small first (default 25).
- `utility_listing_get` takes a `source_id`, an `address` or a `listing_url` (with an optional `source`). Use it for a single known address or URL, and for detail on a keeper.
- `deal_import` returns `requested / created / deduped / failed` and `deals[{source_id, deal_id, deduped, error?}]`. Report the counts, retry only the failed rows.
- Single address or URL, first pass with research: `automation_run_create(goal=import, address | listing_url)`, then poll `automation_run_get`. Do not block on one call.

## Enrich check

Import copies the listing's facts onto the deal. Confirm with `deal_get` (core) that the deal has what the listing showed. Never invent. Never rewrite marketing copy.

| Check | Fix when missing |
|---|---|
| Listing write-up (`listing_description`), verbatim | `deal_update(listing_description)` from `utility_listing_get` or the listing page |
| Photos: the full gallery (not hero-only), no map or satellite images | `deal_item_upsert` with `kind=attachment` rows (`source_url`, `category=listing`, first one `is_primary`). `deal_list(has_gap=photos)` finds deals with none |
| Lat/lng, beds, baths, sqft, year built, lot | `deal_update` property fields (lot sqft ÷ 43,560 = acres) |
| Amenities (closed catalog) and `attributes` (MLS-style facts) | `deal_update(amenities, attributes)`; see the glossary |
| Every portal URL for the house | `deal_item_upsert` with `kind=listing` rows |
| HOA and tax, **only if listed** | `deal_update(hoa_monthly, tax_annual)`; tax history rows are `kind=appraisal_tax` (the **`research`** skill verifies them) |
| Listing agent | `contact_list(q=phone_or_email)`, then `contact_link` with `role=listing_agent`; create the contact with `contact_upsert` (include phone or email so the same person is reused across deals). Skip and say so for FSBO and auction listings with no agent |
| Auction date, liens (distress) | `deal_update(auction_date, liens_owed)`, only when known |

Do **not** treat a Zestimate as rent or ARV. Default status for a new lead is `new` (inbox); do not default an import to `researching`. Do not stuff the lead source into a title.

## After import

- Brief: created and deduped counts, 2–3 standouts, anything still card-only.
- **Standouts by aggressiveness** (the **`realtypad`** skill): LTR/STR → distress or positive yield worth a watch; flip/wholesale → only clear asymmetry.
- **Stop.** Next steps are the **`research`**, **`triage`** or **`underwrite`** skill, only if the user asks.

Example gap-fill call (`kind` goes on each row inside `items`):

```
deal_item_upsert(deal_id, items=[
  {kind: "attachment", source_url: "https://…/photo1.jpg", category: "listing", is_primary: true},
  {kind: "listing", source_url: "https://www.zillow.com/homedetails/…"}
])
```

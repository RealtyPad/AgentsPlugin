---
name: trends
description: >-
  Check and refresh RealtyPad market trends for a deal or cluster. Prefer
  ensure_geo_markets (when geos are missing) then
  refresh_deal_trends(gap_fill=true) as the entrypoint. Power tools
  refresh_catalog_trends / refresh_airroi_trends only when surgical. Tenant
  upsert is for custom/override sources only. Use when the user asks to update
  trends, refresh ZHVI/ZORI/AirROI, read market projections, or when
  research/triage needs fresh market series.
---

# RealtyPad — trends (market snapshots)

## Hard rules

- **Never invent numbers.** Null > fiction. Cite `source_url` on every upsert.
- Percent fields are **fractions**: YoY −3.2% → `-0.032`; occ 55% → `0.55`; unemployment 4.2% → `0.042`.
- `source` must match `^[a-z0-9_]+$`.
- Unique key (tenant): `(market_id, as_of, source)` — re-upsert updates the same point.
- **Split geos from refresh.** Link the chain with **`ensure_geo_markets`**. Decide whether to
  refresh with **`refresh_deal_trends`** (scan / `gap_fill`) or surgical
  **`refresh_catalog_trends`** / **`refresh_airroi_trends`**. Do **not** treat a mega
  “ensure everything” call as the default — agents choose what to refresh and when.
- **Shared catalog:** `zillow_zhvi`, `zillow_zori`, listing/liquidity series
  (`zillow_inventory`, `zillow_new_listings`, `zillow_dom`, `zillow_price_cut`,
  `zillow_median_sale`, `zillow_market_temp`, `zillow_zhvf`), `fred_pmms`,
  `census_bps`, `bls_laus`, `census_pop`, `census_acs_vacancy`, **`airroi` (STR)**
  are platform-owned. **CSV/file series** (Zillow + FRED + BPS + PEP `census_pop`)
  are refreshed weekly by the platform CronJob
  (`python -m app.cli catalog refresh-national`) for **all US regions** in each file.
  Prefer reading merged snapshots + selective gap-fill over hand-calling
  `refresh_catalog_trends` for every source. **Do not** LLM-parse catalog CSVs into
  `upsert_market_snapshots_bulk`. Reads merge catalog automatically (tenant overlay wins).
- **Per-source geography gates** (not one blanket supply set):
  - BPS / LAUS: **county / msa / state** only
  - `census_pop`: county / msa / state **+ zip** (zip via ACS ZCTA, not PEP)
  - `census_acs_vacancy`: county / msa **+ zip** (ZCTA)
  - Listing CSVs: **zip + neighborhood** (ZHVF zip-only)
  - ZORI: zip / city / county / msa (not neighborhood)
  - **AirROI STR:** **zip / neighborhood / city / county** (provider max **60** months;
    state/MSA unsupported — AirROI requires a locality)
- Market `rental_vacancy_pct` / permits / jobs / listing metrics are **advisory only** —
  never overwrite deal `vacancy_pct` or hold growth from these series.
- Prefer official downloads for non-catalog sources (FHFA, Redfin) over fragile scrapes.
  AirROI STR uses server catalog ingest (`refresh_deal_trends(gap_fill=true)` /
  `refresh_airroi_trends`), not LLM CSV/JSON parsing.
- Do **not** change deal status or underwrite verdicts from a trends pass.
- **Geos:** `ensure_geo_markets(deal_id=…)` links neighborhood→zip→city→county→msa→state.
  Pass listing `postal_code` / `city` / `state` / `county` / `msa` / `neighborhood` /
  `street` / `latitude` / `longitude` when known so they **overlay** the catalog reverse
  base. Already-linked neighborhoods are **preserved** when `neighborhood` is omitted.
  Prefer `deal_id` (property street / lat/lng) so ensure can geocode and reverse-lookup
  catalog boundaries — do not hand-edit zip maps. Catalog keys match `(level, code)`.
  County FIPS for catalog ingest come from the vendored Census national county table
  (`us_county_fips.json`), not live geocoding.
- **ZCTA ≠ USPS zip** in edge cases — ACS zip series follow Census ZCTA boundaries;
  never invent vacancy/pop when a ZCTA is missing.
- **Map overlays:** Trends UI draws polygons from platform `catalog_boundaries`
  (Census Cartographic Boundary 500k GeoJSON + CDND neighborhoods). Levels:
  state/msa/county/city/zip/neighborhood. Never invent polygons; if the API 404s,
  leave the map empty / missing state.
- **Geo ensure merge order (`ensure_geo_markets`):**
  1. Resolve lat/lng (request → property → Nominatim); persist when property lacked coords.
  2. Catalog boundary reverse at the point = **base layer**.
  3. Census Geocoder fills blanks only (street forward and/or coords reverse).
  4. Non-empty caller/listing geos **overlay** (listing wins).
  5. Zip/city known maps fill any remaining blanks.
  - Pass **`neighborhood=`** from the listing when known (Zillow region / subdivision);
    already-linked neighborhoods are preserved when omitted.
  - **New / unbuilt streets** often miss TIGER (no address match). That is a Census data
    gap, not a RealtyPad bug. If street fails, ensure property **`lat`/`lng`** are set and
    re-call with `deal_id` — reverse geocode still resolves county + MSA/CBSA.
  - Do **not** invent county/MSA/neighborhood names to work around a miss; fix street
    spelling or geocode coords first, then refresh catalog on the linked codes.
  - Known codes still work for catalog refresh without geocoding (`caldwell_tx`, etc.)
    once the chain exists.

## Catalog vs tenant overlay

| Layer | Write path | Examples |
|-------|------------|----------|
| **Platform catalog** | `refresh_deal_trends(gap_fill=true)` or `refresh_catalog_trends` / `refresh_airroi_trends` | ZHVI/ZORI/listings/FRED + BPS/LAUS/pop/ACS vacancy + **AirROI STR** |
| **Tenant overlay** | `upsert_market_snapshots_bulk` | Redfin, manual overrides (wins on same as_of+source) |

Resolve rule: same geo `(level, code)` + `source` + `as_of` → **tenant wins**; else catalog.
AirROI catalog refreshes **clear tenant `source=airroi`** shadows so the shared baseline shows.

## Freshness

| Series | Stale if |
|--------|----------|
| Home / LTR / listings / FRED | latest `as_of` older than **~35 days** |
| `census_bps` / `bls_laus` / **`airroi`** | **~45 days** |
| `census_pop` / `census_acs_vacancy` | **~400 days** (annual) |
| Redfin (tenant) | ~35d |

**`gap_fill` behavior:** live APIs refresh when stale **or** missing; CronJob CSV
refreshes when **missing** only (merely-stale CSV → `skipped_csv_cron`, CronJob-owned)
unless `force` / `force_csv=true`.

## Depth targets

| Series | Target history |
|--------|----------------|
| ZHVI / ZORI / listings / FRED / Census monthly | up to **~120 mo** (~10y) |
| AirROI STR | up to **60 mo** (provider API max — do not invent older months) |

## Refresh matrix by geography

| Level | Catalog sources to refresh |
|-------|----------------------------|
| **zip** | ZHVI, ZORI, listing/liquidity (+ ZHVF), ACS vacancy, ACS pop (`census_pop`), **airroi** |
| **neighborhood** | ZHVI + listing/liquidity + **airroi** (no ZORI, no ACS, no ZHVF) |
| **city** | ZHVI, ZORI (where published), **airroi** |
| **county / msa / state** | ZHVI, ZORI (if published), BPS, LAUS, PEP pop, ACS vacancy (county/msa); **airroi on county only** (not msa/state) |
| **country/`US`** | FRED PMMS |

## Workflow

1. **`ensure_geo_markets(deal_id=…, listing geos + lat/lng when known)`** — only when the
   geo chain is missing or needs overlay (neighborhood from listing).
2. **Read** `get_deal` → `markets[].latest_by_source` (expect a neighborhood chip when
   linked). Optional: `refresh_deal_trends(deal_id=…)` **scan** (`ingested=false`) for
   `stale_csv[]` / `stale_api[]`.
3. **Refresh only what you need:**
   - `refresh_deal_trends(gap_fill=true)` — selective gap-fill (no geo linking).
     Time-boxed (~40s): when the result has `complete=false`, call again with the
     same arguments until `complete=true` — `pending[]` shows what is left. Jobs
     tried in the last ~15 min that are still stale show in `errors[]` as "tried
     recently"; don't loop on them — move on or refresh that source surgically later.
   - `refresh_catalog_trends(geography_level, code, sources=[one])` — surgical
   - `refresh_airroi_trends(deal_id=…)` — STR-only deal helper
   - **Never** `force=true` from MCP (timeout).
4. Brief geo × series; history via `list_market_snapshots` / `get_latest_market_snapshot`.
   Tenant upsert = custom/Redfin **only**.

`ensure_deal_trends` remains as a **legacy combo** (optional geos + gap-fill). Prefer the
split tools above.

## Progress checklist

- [ ] 1. `ensure_geo_markets` when geos missing (pass listing geos + lat/lng when known)
- [ ] 2. Read `markets[].latest_by_source` (neigh / zip / city / county …)
- [ ] 3. Scan and/or `refresh_deal_trends(gap_fill=true)` / surgical refresh when gaps matter
- [ ] 4. Brief geo × series matrix (catalog vs overlay vs gap)

## Source cheat sheet

| Need | Source | Path |
|------|--------|------|
| Link geos | — | **`ensure_geo_markets`** |
| Freshness scan | — | **`refresh_deal_trends`** (default) |
| Deal gap-fill | all applicable | **`refresh_deal_trends(gap_fill=true)`** |
| Home index | `zillow_zhvi` | gap_fill / `refresh_catalog_trends` |
| LTR rent | `zillow_zori` | gap_fill / `refresh_catalog_trends` (not neighborhood) |
| For-sale inventory | `zillow_inventory` | `raw.inventory` (zip/neighborhood) |
| New listings | `zillow_new_listings` | `raw.new_listings` |
| Days to pending | `zillow_dom` | `median_dom` |
| % price cut | `zillow_price_cut` | `raw.price_cut_pct` (fraction) |
| Median sale (observed) | `zillow_median_sale` | `median_sale_price` (distinct from ZHVI) |
| Market temp | `zillow_market_temp` | `raw.market_temp_index` |
| ZHVI forecast growth | `zillow_zhvf` | `raw.zhvf_growth_pct` (zip only) |
| Mortgage | `fred_pmms` | gap_fill / `refresh_catalog_trends` on country/`US` |
| Permits | `census_bps` | county/msa/state |
| Jobs | `bls_laus` | county/msa/state (live API via gap_fill) |
| Population | `census_pop` | county/msa/state (PEP) **+ zip (ACS ZCTA)** |
| Rental vacancy | `census_acs_vacancy` | county/msa **+ zip (ZCTA)**; needs Census API key |
| STR | `airroi` | gap_fill / `refresh_airroi_trends(deal_id=…)` → zip/neighborhood/city/county (≤60 mo; not state/msa) |
| Custom override | same source code | Tenant upsert (wins on merge) |

## Out of scope

- Inventing older months to pad depth (including past AirROI’s 60-mo cap)
- LLM-parsing catalog series into tenant bulk upsert
- Changing deal triage status from a trends pass
- Copying supply/demand into deal OpEx / hold growth
- BPS/LAUS at zip; neighborhood ZORI; Redfin/ZBP catalog ingest
- Force-refreshing all CronJob CSV on every trends pass
- Bundling geo-link into every refresh call

---
name: trends
description: >-
  Check and refresh RealtyPad market trends for a deal or cluster. Read with
  trend_get (freshness and series), link missing geographies with market_ensure,
  refresh with trend_refresh (gap_fill) and poll job_get when it returns a job.
  Tenant overrides via trend_override_upsert are for custom sources only. Use
  when the user asks to update trends, refresh ZHVI/ZORI/AirROI, read market
  projections, or when research/triage needs fresh market series.
---

# RealtyPad — trends (market snapshots)

## Hard rules

- **Never invent numbers.** Null > fiction. Cite `source_url` on every upsert.
- Percent fields are **fractions**: YoY −3.2% → `-0.032`; occ 55% → `0.55`; unemployment 4.2% → `0.042`.
- `source` must match `^[a-z0-9_]+$`.
- Unique key (tenant): `(market_id, as_of, source)` — re-upsert updates the same point.
- **Split geos from refresh.** Link the chain with **`market_ensure`**. Read freshness with
  **`trend_get`**, then refresh with **`trend_refresh`** (`gap_fill`, or one geography and
  named `sources` when surgical). Do **not** treat a mega “ensure everything” call as the
  default — agents choose what to refresh and when.
- **Shared catalog:** `zillow_zhvi`, `zillow_zori`, listing/liquidity series
  (`zillow_inventory`, `zillow_new_listings`, `zillow_dom`, `zillow_price_cut`,
  `zillow_median_sale`, `zillow_market_temp`, `zillow_zhvf`), `fred_pmms`,
  `census_bps`, `bls_laus`, `census_pop`, `census_acs_vacancy`, **`airroi` (STR)**
  are platform-owned. **CSV/file series** (Zillow + FRED + BPS + PEP `census_pop`)
  are refreshed weekly by the platform CronJob
  (`python -m app.cli catalog refresh-national`) for **all US regions** in each file.
  Prefer reading merged series with `trend_get` + selective gap-fill over naming every
  source in `trend_refresh`. **Do not** LLM-parse catalog CSVs into
  `trend_override_upsert`. Reads merge catalog automatically (tenant overlay wins).
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
  AirROI STR is a catalog source (`trend_refresh` with `sources=["airroi"]`, or gap-fill),
  not LLM CSV/JSON parsing.
- Do **not** change deal status or underwrite verdicts from a trends pass.
- **Geos:** `market_ensure(deal_id=…)` links neighborhood→zip→city→county→msa→state.
  Pass listing `postal_code` / `city` / `state` / `county` / `msa` / `neighborhood` /
  `street` / `latitude` / `longitude` when known so they **overlay** the catalog reverse
  base. Already-linked neighborhoods are **preserved** when `neighborhood` is omitted.
  Prefer `deal_id` (property street / lat/lng) so ensure can geocode and reverse-lookup
  catalog boundaries — do not hand-edit zip maps. Catalog keys match `(level, code)`.
  County FIPS for catalog refresh come from the vendored Census national county table
  (`us_county_fips.json`), not live geocoding.
- **ZCTA ≠ USPS zip** in edge cases — ACS zip series follow Census ZCTA boundaries;
  never invent vacancy/pop when a ZCTA is missing.
- **Map overlays:** Trends UI draws polygons from platform `catalog_boundaries`
  (Census Cartographic Boundary 500k GeoJSON + CDND neighborhoods). Levels:
  state/msa/county/city/zip/neighborhood. Never invent polygons; if the API 404s,
  leave the map empty / missing state.
- **Geo ensure merge order (`market_ensure`):**
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
| **Platform catalog** | `trend_refresh(scope, gap_fill=true)` (or named `sources`) | ZHVI/ZORI/listings/FRED + BPS/LAUS/pop/ACS vacancy + **AirROI STR** |
| **Tenant overlay** | `trend_override_upsert` | Redfin, manual overrides (wins on same as_of+source) |

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

## Tools

| Tool | Does |
|------|------|
| `trend_get(deal_id \| geo_id, series[], horizon_years, include_projected, include_history, history_source, history_limit)` | Read-only. Per linked geography: `latest_by_source` (with `period_count`, `thin_history`), fitted `series` (`annualized_growth_pct`, `fit_quality`; monthly `projected` points only with `include_projected`), optional `history` rows (tenant wins over catalog; `from_catalog` flag), and `freshness` (`fresh`, `stale[]`). A `deal_id` returns every linked geography, finest first. Prefer `series=` with `include_projected=false` unless you need the path; check `fit_quality`. |
| `trend_refresh(scope={deal_id \| geo_id}, sources[], gap_fill)` | Downloads and stores catalog series. `gap_fill` (default true) refreshes only stale or missing series; `gap_fill=false` forces the named `sources` (needs `sources`). Returns `{job, result}`. |
| `trend_override_upsert(snapshots[])` | Tenant snapshots, one object per period (`geo_id`, `source`, `as_of`, metrics). Partial success: `errors[]` lists bad rows by index. |
| `market_ensure(deal_id, …)` / `market_list` | Link or list geographies (a geography is a market row; `geo_id` is the market UUID). |
| `job_get(job_id)` | Poll a running refresh. |

## Workflow

1. **`market_ensure(deal_id=…, listing geos + lat/lng when known)`** — only when the
   geo chain is missing or needs overlay (neighborhood from listing).
2. **Read** `trend_get(deal_id)` → `latest_by_source` per geography (expect a neighborhood
   chip when linked) and `freshness` (`stale[]`).
3. **Refresh only what you need:**
   - `trend_refresh(scope={deal_id}, gap_fill=true)` — selective gap-fill (no geo linking).
     The result is a job. When it is still `running`, poll `job_get(job_id)`. A time-boxed
     partial refresh is `completed` with `result.complete=false` and `result.pending`:
     call again with the same arguments until `complete=true`. Do not loop on sources that
     keep coming back stale; move on or refresh that source surgically later.
   - `trend_refresh(scope={geo_id}, sources=[one], gap_fill=false)` — surgical, one geography
   - `trend_refresh(scope={deal_id}, sources=["airroi"])` — STR-only deal helper
   - **Never** force-refresh a source that is fresh.
4. Brief geo × series; per-period history via `trend_get(include_history=true)`.
   Tenant override (`trend_override_upsert`) = custom/Redfin **only**.

## Progress checklist

- [ ] 1. `market_ensure` when geos missing (pass listing geos + lat/lng when known)
- [ ] 2. `trend_get(deal_id)`: read `latest_by_source` (neigh / zip / city / county …) and `freshness`
- [ ] 3. `trend_refresh(scope={deal_id})` gap-fill (or surgical) when gaps matter; poll `job_get` if it is still running
- [ ] 4. Brief geo × series matrix (catalog vs overlay vs gap)

## Source cheat sheet

| Need | Source | Path |
|------|--------|------|
| Link geos | — | **`market_ensure`** |
| Freshness scan | — | **`trend_get`** (`freshness`) |
| Deal gap-fill | all applicable | **`trend_refresh(scope={deal_id}, gap_fill=true)`** |
| Home index | `zillow_zhvi` | gap_fill / `trend_refresh` with `sources` |
| LTR rent | `zillow_zori` | gap_fill / `trend_refresh` with `sources` (not neighborhood) |
| For-sale inventory | `zillow_inventory` | `raw.inventory` (zip/neighborhood) |
| New listings | `zillow_new_listings` | `raw.new_listings` |
| Days to pending | `zillow_dom` | `median_dom` |
| % price cut | `zillow_price_cut` | `raw.price_cut_pct` (fraction) |
| Median sale (observed) | `zillow_median_sale` | `median_sale_price` (distinct from ZHVI) |
| Market temp | `zillow_market_temp` | `raw.market_temp_index` |
| ZHVI forecast growth | `zillow_zhvf` | `raw.zhvf_growth_pct` (zip only) |
| Mortgage | `fred_pmms` | gap_fill / `trend_refresh(scope={geo_id})` on country/`US` |
| Permits | `census_bps` | county/msa/state |
| Jobs | `bls_laus` | county/msa/state (live API via gap_fill) |
| Population | `census_pop` | county/msa/state (PEP) **+ zip (ACS ZCTA)** |
| Rental vacancy | `census_acs_vacancy` | county/msa **+ zip (ZCTA)**; needs the server's Census integration |
| STR | `airroi` | gap_fill / `trend_refresh(scope={deal_id}, sources=["airroi"])` → zip/neighborhood/city/county (≤60 mo; not state/msa) |
| Custom override | same source code | `trend_override_upsert` (wins on merge) |

## Out of scope

- Inventing older months to pad depth (including past AirROI’s 60-mo cap)
- LLM-parsing catalog series into `trend_override_upsert`
- Changing deal triage status from a trends pass
- Copying supply/demand into deal OpEx / hold growth
- BPS/LAUS at zip; neighborhood ZORI; Redfin/ZBP catalog import
- Force-refreshing all CronJob CSV on every trends pass
- Bundling geo-link into every refresh call

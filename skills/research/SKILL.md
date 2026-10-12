---
name: research
description: >-
  Fill verified RealtyPad deal inputs from public sources (tax, HOA,
  insurance, rent/ADR/ARV, comps, listing history, listing copy, lat/lng,
  photos) without ranking or scenario judgment; refresh stale market trends
  when needed. Use when researching a deal, clearing blocking/soft data_gaps,
  finding comps, enriching before underwrite/scenarios, or when
  triage/underwrite needs evidence gathered first.
---

# RealtyPad — research (evidence only)

## Hard rules

- **Never invent numbers.** Prefer null over fiction. Mark estimates in comments / `raw` (or structured tabs).
- **Trust but verify ask/list price.** Stored money fields and prior automation output are not ground truth. On revisit, compare live portal ask → stored `ask_price` → **patch when different** (price cuts, re-lists, bid moves) and note the change; also refresh listing status when the portal shows it.
- Read **comments + open `action_needed` handoffs + comps** before re-fetching; reuse prior sources. Do not treat observations as a second comment thread.
- **Do not** set triage verdict (`ranked` / `passed`) or aggressiveness judgment — that is the **`underwrite`** skill / the **`triage`** skill.
- Scenario matrix / select / apply belongs to the **`scenarios`** skill (underwrite calls it).
- Patching money fields via `update_deal` **auto-rescores**. That is fine for clearing estimates. **Never** set `status` / Status note (`rationale`) from research — use `update_deal_status` only in triage/UW. Money-only `update_deals` for multi-deal fills.
- Prefer Firecrawl (or the user’s listing/MLS MCP) for web pages. Do not pull from ToS-hostile sites without approval.

## Owns vs does not

| Owns | Does not own |
|------|----------------|
| Tax, HOA, insurance, rent / ADR+occ / ARV+repairs | Status rubric / aggressiveness |
| `listing_description`, lat/lng, photos, amenities, listing URLs (`deal_listings`) | Scenario matrix / runs (the **`scenarios`** skill) |
| Listing history / Time on market (`list_deal_listing_events`, `add_deal_listing_events`) | Scoring / buyer match (TOM is research signal only) |
| Research **comments**, **`action_needed` handoffs**, **structured comps** (`add_deal_comp`), portal source URLs | Pattern-pass clusters (triage) |
| Gap list for underwrite | Bulk market ingest (the **`ingest`** skill) |
| Triggering the **`trends`** skill when market cache is stale | Owning snapshot upsert playbooks (trends workflow) |
| Triggering the **`growth-drivers`** skill for qualitative demand signals | Owning Drivers grading playbook |
| Pointing CapEx work to the **`cost-estimate`** skill | Owning HD Apify BOM runs (cost-estimate workflow) |

## Checklist

```
Research Progress:
- [ ] 1. Load deal (`get_deal` **core**) + `list_comments` / handoffs / `list_deal_comps` + read ``data_gaps`` (any blocking=true → fill before UW)
- [ ] 2. Note strategy money gaps from ``strategy_intent`` when set (else ``income_strategy``): ltr rent | str ADR+occ | flip/wholesale/redevelop ARV (exit/land) — intent can emit multiple (e.g. rent + adr)
- [ ] 3. Market trends: call **`ensure_geo_markets`** when geos missing (pass neighborhood when known), then read `markets[].latest_by_source`; gap-fill with **`refresh_deal_trends(gap_fill=true)`** or surgical refresh when stale/missing to brief (ZHVI/ZORI/AirROI chips + `period_count`). Do **not** tenant-upsert catalog CSVs; do **not** use `refresh_deal_trends` force or LLM-parse ZHVI/ZORI. Power tools (`refresh_catalog_trends` / `refresh_airroi_trends`) only when surgical.
- [ ] 3b. Growth drivers (**required** for import/research ready package): load the **`growth-drivers`** skill — search local news/planning/employers; log ≥1 active driver via `add_deal_growth_factor(s)` (strength×direction + source). Soft gap `missing_growth_drivers` until cleared. Do not confuse with Trends YoY Growth. Do not claim research complete without active drivers.
- [ ] 4. Verify ask / purchase / bid from source_url (label which): compare portal price vs stored `ask_price` / `purchase_price`; patch on price cut or other change; refresh listing status when shown
- [ ] 4b. Listing history / Time on market: scrape portal price history (Zillow `priceHistory` from detail Actor / Firecrawl) → persist with **`add_deal_listing_events`** (or `add_deal_listing_event`). Prefer structured events over freeform notes. Tag past **rent** listings (`Listed for rent`, rows where the ask is a monthly rent) with `listing_type=rental`; they stay in the timeline as a rent signal (e.g. "failed to rent at $2,000") but never count toward Time on market. Sale rows are the default (`listing_type=sale`). Platform derives Time on market / True time on market from sale events — do not patch those scalars directly. Hosted tab **Research listing history** runs a refresh automation (`kind=listing_history`).
- [ ] 5. listing_description if missing — detail page / MLS / Firecrawl (never invent). Ingest should already have copied the write-up; if the deal is still a search card, run the same detail pull as ingest.
- [ ] 5b. Amenities when the listing shows them (`update_deal_property` amenities replace-array — never invent). Attach every portal detail URL found via `add_deal_listing` / ingest `listing_urls` (not only primary `source_url`).
- [ ] 5c. Property **attributes** (Interior/Exterior/parking/subdivision/`mls_id`/etc. from portal detail grids — HAR, Zillow, …): patch `update_deal_property` `attributes` JSONB (top-level shallow merge; null deletes a key). Soft convention keys in the glossary. Hosted listing refresh automations fill attributes the same way. **Do not** build portal feature adapters — research the detail page and patch. HOA/maintenance fee → `update_deal` `hoa_monthly` (or refresh `hoa_monthly` field) — never leave HOA inside attributes.
- [ ] 6. lat/lng (portal/MLS coords preferred; else Nominatim backfill)
- [ ] 7. tax_annual from **assessor history**, not listing quote alone. Prefer `listingTaxHistory` / `_details.taxHistory` already on the Zillow detail payload (persist via `add_deal_appraisal_tax_history`, `sync_deal_tax=true` for the current UW year). County CAD sites often block automation (`robots.txt`) — do not scrape them. If ingest already stored history, reuse it.
- [ ] 8. HOA if condo/HOA community
- [ ] 9. Income inputs: rent comps | ADR+occ | ARV+repairs (null if unknown) — store comps via add_deal_comp; for CapEx use the **`cost-estimate`** skill (HD materials + labor ranges)
- [ ] 10. insurance_annual if a real quote exists; else leave profile/STR default
- [ ] 11. Photos: full gallery vs listing (not “any photo exists”) — see Photos below
- [ ] 12. Patch **money only** via `update_deal` / money-only `update_deals` (never status/rationale); property beds/baths/sqft/year_built/lot/property_type/amenities/`attributes`/address via `update_deal_property`; comps + research comment with sources; `action_needed` handoffs only for next-agent todos
- [ ] 13. Brief: what filled, what still missing, market trend freshness, growth drivers logged (no ranked/passed)
```

## MCP

RealtyPad MCP tools (same API as the hosted app):

| Step | Tool |
|------|------|
| Load | `get_deal` (core), `list_comments`, `list_deal_comps` |
| Market cache | via the **`trends`** skill (`ensure_geo_markets` → scan/gap_fill via `refresh_deal_trends`; surgical: `refresh_catalog_trends` / `refresh_airroi_trends`; tenant upsert only for custom overlays) |
| Patch | `update_deal` / **`update_deals`** (**money** + `listing_description` only — no status/rationale; prefer bulk for multi-deal fills); **`update_deal_property`** (beds/baths/sqft/year_built/lot_size_acres/property_type/amenities/`attributes`/address — not on `update_deal`) |
| Tab research (UI) | Deal detail tabs enqueue **`deal_tab_refresh`** via `POST /api/automation-runs` with `deal_id` + `kind` (`goal=refresh`). Full-deal **Research** / **Underwrite** use the same endpoint with `goal=research|underwrite` (MCP `create_automation_run`). Sync REST remains for listing/trends/appraisal (`…/listing/refresh`, `…/trends/ensure`, `…/appraisal-tax/refresh-from-listing`). Listing history uses a refresh automation (`kind=listing_history`) — not a sync refresh shortcut. Listing research must capture **listing agent** (`link_deal_contact` `role=listing_agent` or `listing_agent` on `add_manual_lead`) when the portal shows attribution — clears soft gap `missing_listing_contact`. Planner goals: `read_doc("agents/automation-planner.md")`. |

**Expenses / comps tab research kinds** (not Quick Analyze):

| UI | `kind` | Playbook |
|----|--------|----------|
| Find comps | `comps` | `research` |
| Research listing history | `listing_history` | `research` |
| Research growth drivers | `growth_drivers` | `growth-drivers` |
| Research fix-up costs | `capex` | `cost-estimate` |
| Research monthly costs | `opex` | `opex` |
| Contacts | `list_deal_contacts` / `link_deal_contact` / `list_contacts` (q matches name **and** phone/email) / `get_contact` / `merge_contacts` / `delete_contact` — listing agent or seller (FSBO). Prefer find-by-channel then `contact_id`; create+link with phone/email auto-reuses. |
| Comps | `list_deal_comps` / `add_deal_comps` (bulk) / `add_deal_comp` / `update_deal_comp` / `delete_deal_comp` (`kind`: rent \| sale \| str). Connectors: `airroi_listings_comparables` (STR), [Zillow](../ingest/references/data-sources/zillow.md) PPE (rent/sale), Firecrawl/web search — catalog: the data-source catalog ([data-sources.md](../ingest/references/data-sources.md)). |
| Notes | `add_comment` (research brief / findings); `add_observation` **only** `kind=action_needed` for next-agent todos; `resolve_observation` when one is handled |
| Photos | `add_deal_attachment_urls` (gallery bulk) / `add_deal_attachment_url` / `delete_deal_attachment` |

**Comps:** There is **no** deal-scoped find+persist shortcut. Discover with web search/browse first, then allowlisted connectors, then persist via `add_deal_comps` / `add_deal_comp`. Hosted Quick Analyze / Comps-tab **Research comps** run a hosted automation that finds and saves comps — that path is intentional; do not recreate a silent find tool for chat agents.

**Strategy scope (cost-critical):** Only search comps kinds implied by **`strategy_intent`** when set, else active **`income_strategy`** (owner-occ `primary` → rent+STR+sale). Map: LTR → `rent`; STR → `str`; flip/wholesale/redevelop → `sale`. Empty investment focus defaults to **rent only** — never “search everything.” Land/tear-down theses: the **`redevelopment`** skill — when `qa_flags.teardown_signals` is non-empty, the condition decides the strategies (redevelop only; no rent / flip ARV just to clear gaps).

| Kind | Tool | When |
|------|------|------|
| `rent` | Zillow PPE / Firecrawl — [zillow.md](../ingest/references/data-sources/zillow.md) | LTR (or primary) |
| `sale` | Zillow PPE / Firecrawl — [zillow.md](../ingest/references/data-sources/zillow.md) | flip / wholesale / redevelop (or primary) |
| `str` | **`airroi_listings_comparables` only** (paid); optional [VRBO](../ingest/references/data-sources/vrbo.md) / [AirDNA](../ingest/references/data-sources/airdna.md) | STR in focus — **never** call AirROI for LTR-only / flip-only / wholesale-only runs |

Always persist **`image_url`** (listing hero) from connector JSON when present — UI thumbnails depend on it. Automation apply also backfills heroes from tool payloads when the model omits them.

1. **Discover** — Firecrawl / session web search for the **in-scope** kinds only. Prefer multiple portals over one source.
2. **Fill gaps** — `airroi_listings_comparables(…)` **only if STR is in scope** (needs `AIRROI_API_KEY`; returns candidates only); Zillow PPE for rent/sale near the subject when those kinds are in scope (needs `APIFY_TOKEN` + lat/lng). Cap spend. Multifamily bed/size rules: [zillow.md](../ingest/references/data-sources/zillow.md).
3. **Persist** — `add_deal_comps` with `source` + `source_url` (upserts by kind+url). `price` = monthly rent (`rent`), list/sale $ (`sale`), or ADR nightly (`str`). Include address, beds/baths/sqft, `image_url` when known. Set `property_type` (single_family, condo, townhouse, duplex, triplex, multi_family, or the portal home type) whenever the listing states it — the comp screen drops a condo/unit against a house only when that type is saved. A unit marker in the address (`#A`, `Unit B`, `Apt`) is stored as an apartment when you omit the type. Never invent.
4. **Patch UW money** only after evidence — do not auto-set rent/ADR/ARV from comps medians alone. For **ARV** on **flip / wholesale**, ARV is an **after-repair** price: derive it from **sale comps only** — median sale $/sqft × subject sqft → soft-filtered sale comps (beds±1 / distance) → raw sale-price median last resort — and only with **≥3 sale comps** (prefer renovated / turnkey closed sales). The county **appraisal / assessed value is an as-is value, never the ARV** — keep it as a cross-check only. For **redevelop** (without flip/wholesale in focus), prefer lot/tear-down/as-is comps and treat the latest appraisal as exit/land value — it leads the method order there (`redevelopment_as_is` residual allowlisted). The hosted fill follows the same rules and re-derives an ARV it set itself when comps change. When methods disagree by ≥5%, leave `arv` null (hosted fill records `raw.automation_uw.arv_skip` with a **code-inferred** residual_reason — agents cannot set it via MCP). After one unexplained skip, stop retrying. Call **`estimate_arv(deal_id)`** (read-only) to see every method's value, which one the fill would pick, and why it would skip — then patch `arv` yourself only if you agree. **Comp quality:** ARV and rent fills (and their validators) use only comps that pass the comp screen — dropped for `property_type_mismatch`, `size_mismatch` (outside 0.65–1.5× subject sqft), `beds_mismatch` (±2+), `too_far` (>10 mi), `stale` (sold/listed >12 mo), `future_date` (as_of after today), or `price_outlier` ($/sqft outside 0.6–1.67× the kept median, ≥4 comps). Checks only fire when both sides carry the fact. See drops via **`screen_deal_comps(deal_id, kind)`** or `raw.automation_uw.comp_screen`; when you have looked at a comp and disagree, pin it with `update_deal_comp(quality_override=keep|exclude|clear)`. Replace dropped comps with better ones rather than overriding by default. Thin/no comps after one reasonable attempt → switch method ($/sqft, specialty); do not widen forever; do **not** “try STR next” unless STR is in `strategy_intent`.
5. **Self-check with the automation's own tools** (all read-only): **`preview_uw_fill(deal_id)`** shows what the hosted fill would set on null rent / ADR+occ / ARV / vacancy / insurance / HOA and from what source; **`check_uw_values(deal_id, estimated_rent=…, arv=…)`** runs the checks that make automation clear a value (`outside_basis_band`, `copied_single_comp`, `unadjusted_size_bias`, `unexplained_method_residual`, `disagrees_with_priority_method`, …) — check before `update_deal`; **`get_deal_qa_flags(deal_id)`** returns the critic's QA flags (distress terms, repairs disclosed but no fix-up costs, foreign listing URLs, thin comps, amenity contradictions, money keys on attributes). Clear what they flag before calling research done.
6. Market-level STR series still go through the **`trends`** skill / `refresh_airroi_trends` (catalog trends ≠ paid listings comps).

## Expenses mental model (CapEx vs OpEx)

| Bucket | Persist how | Feeds | Never |
|--------|-------------|-------|-------|
| **Acquisition CapEx** | `add_deal_repair_item(s)` via the **`cost-estimate`** skill (scalar only if no items) | `repair_estimate` → `cash_in`; sets `needs_rehab` | Invent into OpEx / bump `maintenance_pct` |
| **Recurring OpEx** | `add_deal_opex_item(s)` via the **`opex`** skill | `monthly_opex_total` → monthly CF | Day-0 rehab BOM |

**Never** use `maintenance_pct` / `insurance_annual` as a blended CapEx or OpEx proxy. Do not add a line that duplicates the hard-coded STR 20% platform fee. Soft gap: `needs_rehab` + null estimate → `repairs_unknown` — flag it; do not invent OpEx. CapEx is **not** subtracted from monthly LTR/STR CF, but **is** in `cash_in` (down + repairs) when set. When repair items exist, they own `repair_estimate` (scalar patches ignored).

Money fields: `ask_price`, `purchase_price`, `estimated_rent`, `adr_nightly`, `occupancy_pct`, `arv`, `repair_estimate`, `needs_rehab`, `income_verified` (STR ADR/occ confirmed — never from comps medians alone), `tax_annual`, `insurance_annual`, `hoa_monthly`, `vacancy_pct`, `maintenance_pct`, `hold_months`.

For AS-IS / incomplete reno listings: set `needs_rehab=true` (do not invent CapEx). When `needs_rehab` and `repair_estimate` is null, scoring sets `raw.cashflow.repairs_unknown` + low confidence and the deal appears in Soft gaps (`has_soft_gap`) — same soft-bar as rent capped.

## What “verified” means

| Field | Acceptable sources |
|-------|-------------------|
| Tax | **Assessor history first:** Zillow detail `listingTaxHistory` / `_details.taxHistory` (already fetched at ingest — [zillow.md](../ingest/references/data-sources/zillow.md)), HUD/Auction docs, or county CAD **when the site allows**. Listing-quoted tax is **not** final — it can be ±20–35% off (homestead cap, exemptions). Store multi-year via `add_deal_appraisal_tax_history` (bulk) or `add_deal_appraisal_tax`; set UW with `sync_deal_tax_as_of` / `sync_deal_tax=true` — the platform **derives post-sale tax** (ask/purchase × median effective rate from history) and keeps it **estimated**. Never copy seller `taxPaid` with exemptions as verified. If the latest year jumps **>15%**, flag likely homestead-cap removal. County CAD (`traviscad.org`, etc.) often `ROBOTS_DISALLOWED` — respect robots.txt; do not work around. |
| HOA | Listing, HOA docs, builder page, MLS |
| Rent | Listed rent, lease, comps; Zestimate = estimate |
| ADR / occ | Airbnb/VRBO comps, [VRBO](../ingest/references/data-sources/vrbo.md) geo, [AirDNA](../ingest/references/data-sources/airdna.md) on known Airbnb URLs, market reports — never invent |
| ARV | Flip/wholesale: ≥3 sale comps — $/sqft×subject, then filtered sale comps; raw median last resort; appraisal is as-is, cross-check only. Screened comps only — check `estimate_arv` / `screen_deal_comps`. Redevelop: appraisal (exit/land value) first. Mark estimate; ≥5% method residual → explain or leave null |
| Insurance | Quote or leave profile/STR default (flag estimated) |
| Purchase | Ask, starting bid, or explicit offer — label which |
| Listing description | Portal / MLS remarks — never invent |
| Lat / lng | Portal/MLS coords preferred; Nominatim fallback; never invent |
| Photos | Full listing gallery when the portal shows N images — not hero-only |

## Photos (gallery completeness)

Check **`data_gaps`** on `get_deal` (core) / `list_deals` first — `{ code: "photos", blocking: true }` when `photo_count==0` and `status != new`. Batch: `list_deals(has_gap="photos")` or `has_blocking_gap=true`. Also `list_deal_attachments` → count `kind=image` vs the listing’s photo count (“See all N photos”, Auction.com “N Photos”, MLS media count).

**Quick Analyze / auto-import / gallery refresh:** call the detail Actor then **`get-dataset-items`** (not Firecrawl alone) and persist the full URL list (cap ~40) with `add_deal_attachment_urls`. A hero or 4–5 mini-card sample is not a gallery. Prefer portal CDN / Actor gallery arrays; Firecrawl is fallback only when the dataset has no gallery. **`goal=refresh` (deal-tab Photos / Listing)** always re-fetches. After photos are attached, hosted automation runs a **vision keep/drop** pass (subject interiors/exteriors kept; faces, other homes, maps, logos detached).

| Situation | Action |
|-----------|--------|
| Blocking `photos` gap / zero images | Attach full gallery from Firecrawl/`source_url` or connector detail payload — **do this**, do not only leave a “missing gallery” note |
| Some images but **fewer than listing N** (within cap ~40) | **Backfill the missing ones** — do not stop because a primary already exists |
| Connector search-only had 1 `imgSrc` / empty `allPropertyPhotos` | Re-pull Zillow detail with **`addresses`** ([zillow.md](../ingest/references/data-sources/zillow.md)); Firecrawl carousel URLs → prefer `-p_f.jpg` hashes; Auction.com/imgix gallery |

Rules:

1. First image `is_primary=true` if none yet; otherwise keep existing primary.
2. Skip map/satellite Google URLs, logos, and exact `source_url` dupes (same CDN path or same photo hash).
3. Note in research comment if gallery still short of listing N after best effort.
4. To-dos for the next agent use observation `kind=action_needed` (handoffs only — not a research diary). “Missing gallery…” as a passive finding is wrong when photos are still attachable.
5. Any `data_gaps` entry with `blocking: true` means do not advance to score/UW until cleared.

Ingest owns first-pass galleries (the **`ingest`** skill); research **owns the gap-fill** when triage/UW hits a thin gallery / blocking `photos` gap.

Travis protest band (user-owned nearby): listed tax is not permanent; note ~$700–$1,200/yr when relevant.

## Connector preference (research fetch)

Use whatever the session has, in order:

1. **Deal `source_url`** via Firecrawl (or user listing MCP)
2. **County / assessor / HOA** public pages
3. **User MLS MCP** when connected and the listing is MLS-backed
4. **Allowlisted detail Actor** only when cheaper portals fail, spend is approved — recipes in the data-source catalog ([data-sources.md](../ingest/references/data-sources.md)) (Zillow: **`addresses`**, not bare zpids / not `searchResultsDatasetId`). Do **not** use `apify/rag-web-browser` or other non-allowlisted Store Actors.

Do not require paid connectors for research (except known-URL [AirDNA](../ingest/references/data-sources/airdna.md) enrich when you already have Airbnb room URLs and need listing-level ADR/occ).

## Comps (multi-source) + optional STR enrich

**Nearby comps** (deal needs lat/lng for geo tools):

1. Web search / Firecrawl for rent, sold, and STR listings (multi-portal when possible)
2. `airroi_listings_comparables` for STR candidates near the subject → review → `add_deal_comps(kind=str, source=airroi)`
3. Zillow PPE for rent/sale → map → `add_deal_comps` (`source=zillow`) — [zillow.md](../ingest/references/data-sources/zillow.md)
4. Optional STR geo: [VRBO](../ingest/references/data-sources/vrbo.md) (comps only — not sale ingest). Cap `maxTotalChargeUsd`.
5. Review comps; then `update_deal` with rent / ADR+occ / ARV only after evidence (do not auto-patch from comps). Thin/no comps after one reasonable attempt is a **market limit** — switch valuation method; do not keep re-querying LTR.
6. Market-level STR series still go through the **`trends`** skill / `refresh_airroi_trends(deal_id=…)` / catalog `airroi`

**Known Airbnb URL enrich** (optional): [airdna.md](../ingest/references/data-sources/airdna.md). Prefer AirROI / VRBO / web STR comps as market finders. **Booking.com is not allowlisted.** Never invent ADR/occ.

## Comment template (research-only)

```markdown
## Research — YYYY-MM-DD

**Strategy:** ltr | str | fix_flip | wholesale
**Purchase assumption:** $X (ask | bid | offer)
**Tax:** $Y/yr (source) | still missing
**HOA:** $Z/mo or none | still missing
**Income:** rent $R / ADR $A @ occ O% / ARV $V (source; estimate?)
**Market:** source / as_of / median or ADR (fresh | refreshed | still stale)
**Still missing:** …
**Sources:** URL1, URL2
```

## Brief the user

- What was filled vs still null / estimated.
- Market trend freshness (and whether trends workflow ran).
- One next step: underwrite, triage, or more research — **do not** declare ranked/passed unless the user also asked for underwrite.

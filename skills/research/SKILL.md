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
- Patching money fields with `deal_update` **auto-rescores**. That is fine for clearing estimates. **Never** send `status` / Status note (`rationale`) from research: a status row belongs to triage and UW. Money-only `deal_update` rows for multi-deal fills (one call, one row per deal).
- **Search with `utility_*`, then save by `source_id`.** The `utility_*` tools only read and return `source_id`s; saving is `deal_item_upsert(source_ids)` or `deal_update(property_source_id)`. Public data only. The server writes the cached rows, so you never retype them.
- Prefer Firecrawl (or the user’s listing/MLS MCP) for web pages the `utility_*` tools do not cover. Do not pull from ToS-hostile sites without approval.

## Owns vs does not

| Owns | Does not own |
|------|----------------|
| Tax, HOA, insurance, rent / ADR+occ / ARV+repairs | Status rubric / aggressiveness |
| `listing_description`, lat/lng, photos, amenities, listing URLs (`kind=listing` rows) | Scenario matrix / runs (the **`scenarios`** skill) |
| Listing history / Time on market (`kind=listing_event` rows) | Scoring / buyer match (TOM is research signal only) |
| Research **comments**, **`action_needed` handoffs**, **structured comps** (`kind=comp` rows), portal source URLs | Pattern-pass clusters (triage) |
| Gap list for underwrite | Bulk market import (the **`import`** skill) |
| Triggering the **`trends`** skill when market cache is stale | Owning market-series refresh playbooks (trends workflow) |
| Triggering the **`growth-drivers`** skill for qualitative demand signals | Owning Drivers grading playbook |
| Pointing CapEx work to the **`cost-estimate`** skill | Owning materials price pulls (cost-estimate workflow) |

## Checklist

```
Research Progress:
- [ ] 1. Load deal (`deal_get` **core**) + `comment_list` / handoffs / `deal_item_list(kind=comp)` + read ``data_gaps`` (any blocking=true → fill before UW)
- [ ] 2. Note strategy money gaps from ``strategy_intent`` when set (else ``income_strategy``): ltr rent | str ADR+occ | flip/wholesale/redevelop ARV (exit/land) — intent can emit multiple (e.g. rent + adr). `utility_property_get(deal_id)` shows every source's value for a fact, where they agree, and what is still missing
- [ ] 3. Market trends: `trend_get(deal_id)` for `latest_by_source` and `freshness`; `market_ensure` when geos are missing (pass neighborhood when known); `trend_refresh(scope={deal_id})` (gap_fill) when stale or missing (ZHVI/ZORI/AirROI chips + `period_count`). Do **not** upsert catalog CSVs as overrides; do **not** force-refresh fresh series. Overrides (`trend_override_upsert`) are for custom sources only.
- [ ] 3b. Growth drivers (**required** for import/research ready package): load the **`growth-drivers`** skill — `utility_growth_driver_search` → save the ones that fit with `deal_item_upsert(kind=growth_driver, source_ids)`; log ≥1 active driver. Soft gap `missing_growth_drivers` until cleared. Do not confuse with Trends YoY Growth. Do not claim research complete without active drivers.
- [ ] 4. Verify ask / purchase / bid from source_url (label which): `utility_listing_get(listing_url)` or the portal page; compare portal price vs stored `ask_price` / `purchase_price`; patch on price cut or other change; refresh listing status when shown
- [ ] 4b. Listing history / Time on market: scrape the portal price history (Firecrawl or the detail page) → persist as `deal_item_upsert` rows with `kind=listing_event` (`event_kind`, `event_at`, `ask_price`, `listing_type`, `source_url`). Prefer structured events over freeform notes. Tag past **rent** listings (`Listed for rent`, rows where the ask is a monthly rent) with `listing_type=rental`; they stay in the timeline as a rent signal (e.g. "failed to rent at $2,000") but never count toward Time on market. Sale rows are the default (`listing_type=sale`). Platform derives Time on market / True time on market from sale events — do not patch those scalars directly. Hosted tab **Research listing history** runs a refresh automation (`kind=listing_history`).
- [ ] 5. listing_description if missing — `utility_listing_get` / MLS / Firecrawl (never invent). Import should already have copied the write-up; if the deal is still a search card, run the same detail read as the **`import`** skill.
- [ ] 5b. Amenities when the listing shows them (`deal_update` amenities replace-array — never invent). Attach every portal detail URL found as `kind=listing` rows (not only primary `source_url`).
- [ ] 5c. Property facts and **attributes**: `utility_property_get(deal_id)` merges listing and county records; when it fills gaps, save with `deal_update(deal_id, property_source_id)` (fields in the row win). Interior/Exterior/parking/subdivision/`mls_id`/etc. from portal detail grids go in `deal_update` `attributes` (top-level shallow merge; null deletes a key). Soft convention keys in the glossary. **Do not** build portal feature adapters. HOA/maintenance fee → `deal_update` `hoa_monthly` — never leave HOA inside attributes.
- [ ] 6. lat/lng (portal/MLS coords preferred; else `market_ensure` geocodes from the street)
- [ ] 7. tax_annual from **assessor history**, not listing quote alone. `utility_property_get` returns county records; the listing's tax history is also in the detail. Persist as `kind=appraisal_tax` rows (`as_of`, `tax_annual`, `appraisal_value`); put `sync_deal_tax=true` on the one row that should drive the current UW year. County CAD sites often block automation (`robots.txt`) — do not scrape them. If import already stored history, reuse it.
- [ ] 8. HOA if condo/HOA community
- [ ] 9. Income inputs: rent comps | ADR+occ | ARV+repairs (null if unknown) — `utility_comp_search` → `deal_item_upsert(kind=comp, source_ids)`; for CapEx use the **`cost-estimate`** skill (materials + labor ranges)
- [ ] 10. insurance_annual if a real quote exists; else leave profile/STR default
- [ ] 11. Photos: full gallery vs listing (not “any photo exists”) — see Photos below
- [ ] 12. Patch **money and property** with `deal_update` (never status/rationale in the same row); comps + research comment with sources; `action_needed` handoffs only for next-agent todos
- [ ] 13. Brief: what filled, what still missing, market trend freshness, growth drivers logged (no ranked/passed)
```

## MCP

RealtyPad MCP tools (same API as the hosted app):

| Step | Tool |
|------|------|
| Load | `deal_get` (core), `comment_list`, `deal_item_list(kind=comp)` |
| Property facts | `utility_property_get(deal_id \| address \| listing_url)` → `deal_update(deal_id, property_source_id)` |
| Market cache | via the **`trends`** skill (`market_ensure` → `trend_get` freshness → `trend_refresh`; overrides: `trend_override_upsert`) |
| Patch | `deal_update(deals[])`: one row per deal; **money** + `listing_description` + property fields (beds/baths/sqft/year_built/lot_size_acres/property_type/amenities/`attributes`/address). Never a status field. |
| Tab research (UI) | Deal detail tabs enqueue a refresh via `automation_run_create(deal_id, kind, goal=refresh)`. Full-deal **Research** / **Underwrite** use `goal=research|underwrite`. Poll `automation_run_get`. Listing history uses a refresh automation (`kind=listing_history`). Listing research must capture the **listing agent** (`contact_link` `role=listing_agent`) when the portal shows attribution — clears soft gap `missing_listing_contact`. Planner goals: the RealtyPad docs page `agents/automation-planner`. |

**Expenses / comps tab research kinds** (not Quick Analyze):

| UI | `kind` | Playbook |
|----|--------|----------|
| Find comps | `comps` | `research` |
| Research listing history | `listing_history` | `research` |
| Research growth drivers | `growth_drivers` | `growth-drivers` |
| Research fix-up costs | `capex` | `cost-estimate` |
| Research monthly costs | `opex` | `opex` |
| Contacts | `contact_list` (`deal_id`; `q` matches name **and** phone/email) / `contact_get` / `contact_upsert` / `contact_link` / `contact_merge` / `contact_delete` — listing agent or seller (FSBO). Prefer find-by-channel then `contact_id`; create+link with phone/email auto-reuses. |
| Comps | `deal_item_list(kind=comp, comp_type?)` / `utility_comp_search` → `deal_item_upsert(kind=comp, source_ids)` / `deal_item_upsert` (comp rows; `comp_type`: rent \| sale \| str) / `deal_item_delete(kind=comp)`. Sources: see the table below. |
| Notes | `comment_create` (research brief / findings); `observation_upsert` **only** for `kind=action_needed` next-agent todos; the same tool with `resolved=true` when one is handled |
| Photos | `deal_item_upsert` rows with `kind=attachment` (gallery) / `deal_item_delete(kind=attachment)` |

**Write shapes:** hand-collected rows go in `items`, with `kind` on each row (`deal_item_upsert(deal_id, items=[{kind: "listing_event", event_at: "2026-03-02", event_kind: "price_change", ask_price: 389000, listing_type: "sale", source_url: "https://…"}])`). Search results are saved by id instead: `deal_item_upsert(deal_id, source_ids=[…])`, with the optional top-level `kind` only as a guard that every id is that kind.

**Comps:** discover with `utility_comp_search`, review the items, then save the ones you trust with `deal_item_upsert(deal_id, kind=comp, source_ids=[…])`. Nothing is saved by the search. Web search and Firecrawl comps you find yourself are saved as `kind=comp` rows (`items[]`). Hosted Quick Analyze and the Comps-tab **Research comps** run an automation that finds and saves comps; that path is intentional.

**Strategy scope (cost-critical):** Only search comps kinds implied by **`strategy_intent`** when set, else active **`income_strategy`** (owner-occ `primary` → rent+STR+sale). Map: LTR → `rent`; STR → `str`; flip/wholesale/redevelop → `sale`. Empty investment focus defaults to **rent only** — never “search everything.” Land/tear-down theses: the **`redevelopment`** skill — when the `qa_flags` check (`uw_check`) shows `teardown_signals`, the condition decides the strategies (redevelop only; no rent / flip ARV just to clear gaps).

| `kind` | `utility_comp_search` source (default first) | When |
|------|------|------|
| `rent` | `zillow` | LTR (or primary) |
| `sale` | `zillow` | flip / wholesale / redevelop (or primary) |
| `str` | `airroi` (default), `vrbo` | STR in focus — **never** search `str` for LTR-only / flip-only / wholesale-only runs |

A source that does not serve the kind is `unsupported_source` (the error lists the supported ones). `airdna` cannot search by address. A rent search with neither `sqft` nor `baths` is `rent_subject_unknown`: give the subject's size (or pass `deal_id` to fill beds, baths, sqft and type from the deal). Sale and str searches without size run and warn that comps are not size-filtered. Long searches may return a `job_id`: poll `job_get`. Each search is one paid provider run, cached about a week; `refresh=true` re-runs it, so use it only with a reason.

Comps carry their hero `image_url`, `property_type` and size from the search; keep them when you save rows yourself. Set `property_type` (single_family, condo, townhouse, duplex, triplex, multi_family, or the portal home type) whenever the listing states it — the comp screen drops a condo/unit against a house only when that type is saved. A unit marker in the address (`#A`, `Unit B`, `Apt`) is stored as an apartment when you omit the type. Never invent.

1. **Discover** — `utility_comp_search` for the **in-scope** kinds only (the subject address, plus beds/baths/sqft/type or `deal_id`). Add Firecrawl / web search for another portal when the result is thin. Prefer multiple sources over one.
2. **Review and persist** — drop comps that clearly do not fit, then `deal_item_upsert(deal_id, kind=comp, source_ids)`; dedupe is by URL (`action: deduped`). `price` = monthly rent (`rent`), list/sale $ (`sale`), or ADR nightly (`str`). All ids are written or none: on `source_expired`, search again. Comps you collected by hand: `deal_item_upsert(items=[{kind: "comp", comp_type, price, address, source, source_url, …}])`.
3. **Patch UW money** only after evidence — do not auto-set rent/ADR/ARV from comps medians alone. For **ARV** on **flip / wholesale**, ARV is an **after-repair** price: derive it from **sale comps only** — median sale $/sqft × subject sqft → soft-filtered sale comps (beds±1 / distance) → raw sale-price median last resort — and only with **≥3 sale comps** (prefer renovated / turnkey closed sales). The county **appraisal / assessed value is an as-is value, never the ARV** — keep it as a cross-check only. For **redevelop** (without flip/wholesale in focus), prefer lot/tear-down/as-is comps and treat the latest appraisal as exit/land value — it leads the method order there (`redevelopment_as_is` residual allowlisted). The hosted fill follows the same rules and re-derives an ARV it set itself when comps change. When methods disagree by ≥5%, leave `arv` null (hosted fill records `raw.automation_uw.arv_skip` with a **code-inferred** residual_reason — agents cannot set it via MCP). After one unexplained skip, stop retrying. Call **`uw_check(deal_id, checks=["arv"])`** (read-only) to see every method's value, which one the fill would pick, and why it would skip — then patch `arv` yourself only if you agree. **Comp quality:** ARV and rent fills (and their validators) use only comps that pass the comp screen — dropped for `property_type_mismatch`, `size_mismatch` (outside 0.65–1.5× subject sqft), `beds_mismatch` (±2+), `too_far` (>10 mi), `stale` (sold/listed >12 mo), `future_date` (as_of after today), or `price_outlier` ($/sqft outside 0.6–1.67× the kept median, ≥4 comps). Checks only fire when both sides carry the fact. See drops via **`uw_check(deal_id, checks=["comp_screen"], comp_kind=sale|rent)`** or `raw.automation_uw.comp_screen`; when you have looked at a comp and disagree, pin it with `deal_item_upsert(items=[{kind: "comp", id, quality_override: "keep"|"exclude"|"clear"}])`. Replace dropped comps with better ones rather than overriding by default. Thin/no comps after one reasonable attempt → switch method ($/sqft, specialty); do not widen forever; do **not** “try STR next” unless STR is in `strategy_intent`.
4. **Self-check with the automation's own checks** (all in the one read-only tool **`uw_check`**, one section per check; omit `checks` to run all): `fill_preview` shows what the hosted fill would set on null rent / ADR+occ / ARV / vacancy / insurance / HOA and from what source; `values` (with `estimated_rent=…`, `arv=…`) runs the checks that make automation clear a value (`outside_basis_band`, `copied_single_comp`, `unadjusted_size_bias`, `unexplained_method_residual`, `disagrees_with_priority_method`, …) — check before `deal_update`; `qa_flags` returns the critic's QA flags (distress terms, repairs disclosed but no fix-up costs, foreign listing URLs, thin comps, amenity contradictions, money keys on attributes). Clear what they flag before calling research done.
5. Market-level STR series still go through the **`trends`** skill (`trend_refresh` with `sources=["airroi"]`; catalog trends ≠ paid listing comps).

## Expenses mental model (CapEx vs OpEx)

| Bucket | Persist how | Feeds | Never |
|--------|-------------|-------|-------|
| **Acquisition CapEx** | `deal_item_upsert(items=[{kind: "repair", …}])` via the **`cost-estimate`** skill (scalar only if no items) | `repair_estimate` → `cash_in`; sets `needs_rehab` | Invent into OpEx / bump `maintenance_pct` |
| **Recurring OpEx** | `deal_item_upsert(items=[{kind: "opex", …}])` via the **`opex`** skill | `monthly_opex_total` → monthly CF | Day-0 rehab BOM |

**Never** use `maintenance_pct` / `insurance_annual` as a blended CapEx or OpEx proxy. Do not add a line that duplicates the hard-coded STR 20% platform fee. Soft gap: `needs_rehab` + null estimate → `repairs_unknown` — flag it; do not invent OpEx. CapEx is **not** subtracted from monthly LTR/STR CF, but **is** in `cash_in` (down + repairs) when set. When repair items exist, they own `repair_estimate` (scalar patches ignored).

Money fields: `ask_price`, `purchase_price`, `estimated_rent`, `adr_nightly`, `occupancy_pct`, `arv`, `repair_estimate`, `needs_rehab`, `income_verified` (STR ADR/occ confirmed — never from comps medians alone), `tax_annual`, `insurance_annual`, `hoa_monthly`, `vacancy_pct`, `maintenance_pct`, `hold_months`.

For AS-IS / incomplete reno listings: set `needs_rehab=true` (do not invent CapEx). When `needs_rehab` and `repair_estimate` is null, scoring sets `raw.cashflow.repairs_unknown` + low confidence and the deal appears in Soft gaps (`has_soft_gap`) — same soft-bar as rent capped.

## What “verified” means

| Field | Acceptable sources |
|-------|-------------------|
| Tax | **Assessor history first:** county records from `utility_property_get`, the listing's tax history, HUD/Auction docs, or county CAD **when the site allows**. Listing-quoted tax is **not** final — it can be ±20–35% off (homestead cap, exemptions). Store multi-year history as `deal_item_upsert` rows with `kind=appraisal_tax`; send `sync_deal_tax=true` on the one row that should set the UW tax — the platform **derives post-sale tax** (ask/purchase × median effective rate from history) and keeps it **estimated**. Never copy seller `taxPaid` with exemptions as verified. If the latest year jumps **>15%**, flag likely homestead-cap removal. County CAD (`traviscad.org`, etc.) often `ROBOTS_DISALLOWED` — respect robots.txt; do not work around. |
| HOA | Listing, HOA docs, builder page, MLS |
| Rent | Listed rent, lease, comps; Zestimate = estimate |
| ADR / occ | STR comps (`utility_comp_search(kind=str)`: `airroi`, `vrbo`), market reports — never invent |
| ARV | Flip/wholesale: ≥3 sale comps — $/sqft×subject, then filtered sale comps; raw median last resort; appraisal is as-is, cross-check only. Screened comps only — check `uw_check` (`arv`, `comp_screen``. Redevelop: appraisal (exit/land value) first. Mark estimate; ≥5% method residual → explain or leave null |
| Insurance | Quote or leave profile/STR default (flag estimated) |
| Purchase | Ask, starting bid, or explicit offer — label which |
| Listing description | Portal / MLS remarks — never invent |
| Lat / lng | Portal/MLS coords preferred; Nominatim fallback; never invent |
| Photos | Full listing gallery when the portal shows N images — not hero-only |

## Photos (gallery completeness)

Check **`data_gaps`** on `deal_get` (core) / `deal_list` first — `{ code: "photos", blocking: true }` when `photo_count==0` and `status != new`. Batch: `deal_list(has_gap="photos")` or `has_blocking_gap=true`. Also `deal_item_list(kind=attachment)` → count image rows vs the listing’s photo count (“See all N photos”, Auction.com “N Photos”, MLS media count).

**Quick Analyze / auto-import / gallery refresh:** read the detail with `utility_listing_get` and persist the full URL list (cap ~40) as `deal_item_upsert` rows with `kind=attachment` (`category=listing`, first one `is_primary`). A hero or 4–5 mini-card sample is not a gallery. Prefer portal CDN gallery arrays; Firecrawl is the fallback when the detail has no gallery. **`goal=refresh` (deal-tab Photos / Listing)** always re-fetches. After photos are attached, hosted automation runs a **vision keep/drop** pass (subject interiors/exteriors kept; faces, other homes, maps, logos detached). You can run the same pass yourself with `utility_image_analyze(purpose=gallery, deal_id)` (read-only; at most 12 images; it lists `keep[]` and `drop[]`), then `deal_item_delete(kind=attachment)` the drops.

| Situation | Action |
|-----------|--------|
| Blocking `photos` gap / zero images | Attach the full gallery from `utility_listing_get`, Firecrawl or the `source_url` — **do this**, do not only leave a “missing gallery” note |
| Some images but **fewer than listing N** (within cap ~40) | **Backfill the missing ones** — do not stop because a primary already exists |
| The search card had 1 image / the detail came back thin | Re-read with `utility_listing_get(listing_url, refresh=true)`; Firecrawl carousel URLs → prefer `-p_f.jpg` hashes; Auction.com/imgix gallery |

Rules:

1. First image `is_primary=true` if none yet; otherwise keep existing primary.
2. Skip map/satellite Google URLs, logos, and exact `source_url` dupes (same CDN path or same photo hash).
3. Note in research comment if gallery still short of listing N after best effort.
4. To-dos for the next agent use observation `kind=action_needed` (handoffs only — not a research diary). “Missing gallery…” as a passive finding is wrong when photos are still attachable.
5. Any `data_gaps` entry with `blocking: true` means do not advance to score/UW until cleared.

Import owns first-pass galleries (the **`import`** skill); research **owns the gap-fill** when triage/UW hits a thin gallery / blocking `photos` gap.

Travis protest band (user-owned nearby): listed tax is not permanent; note ~$700–$1,200/yr when relevant.

## Fetch preference (research)

Use whatever the session has, in order:

1. **`utility_*` tools** (`utility_listing_get`, `utility_property_get`, `utility_comp_search`) — cached, metered, and savable by `source_id`
2. **Deal `source_url`** via Firecrawl (or user listing MCP)
3. **County / assessor / HOA** public pages
4. **User MLS MCP** when connected and the listing is MLS-backed

Do not require paid sources for research. Every `utility_*` call is a provider run unless the answer is cached, so read `cached` and `fetched_at` on the result before you pass `refresh=true`.

## Comps (multi-source)

**Nearby comps:**

1. `utility_comp_search` for rent, sale and STR (see the table above) → review → `deal_item_upsert(kind=comp, source_ids)`
2. Web search / Firecrawl for another portal when the result is thin → save as `kind=comp` rows
3. Review comps; then `deal_update` with rent / ADR+occ / ARV only after evidence (do not auto-patch from comps). Thin/no comps after one reasonable attempt is a **market limit** — switch valuation method; do not keep re-querying LTR.
4. Market-level STR series still go through the **`trends`** skill (`trend_refresh(scope={deal_id}, sources=["airroi"])`)

**Booking.com is not a source.** Never invent ADR/occ.

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

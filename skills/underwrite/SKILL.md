---
name: underwrite
description: >-
  Full underwrite of a RealtyPad cashflow, flip, wholesale or redevelop deal:
  run research for verified inputs if needed, prefer advance_deal when score-
  ready, choose which linked geo feeds the UW worksheet / hold growth when
  multi-geo trends exist, run scenarios for persisted scenario runs, apply the
  buyer-fit gate, set triage status, leave an agent comment. Use when the user
  asks to underwrite, UW, fully underwrite, dig into a deal, advance a deal,
  or clear a new / researching / blocked lead.
---

# RealtyPad — underwrite (judgment)

## Hard rules

- **Never invent numbers.** Prefer null over fiction. Mark estimates in comments / `raw`.
- Read **comments + open `action_needed` handoffs** (and `list_scenario_runs`) before recommending next steps — not a second observation diary.
- If money/listing inputs are missing or still profile-estimated, **run the `research` skill first** (or confirm the user already did).
- **Market geo for UW inputs:** when linked `markets[]` has more than one geography with trends, **ask the user which geo level to use** before inserting market-derived values into the underwriting worksheet / hold projections (see below). Do not silently pick MSA vs zip vs neighborhood.
- Score via strategy models (`apply_strategy`):
  - `ltr` → **`cf_v1`** (exposes `monthly_cashflow`, `monthly_noi`, `annual_noi`, `cap_rate_pct`)
  - `str` → **`str_v1`** (needs `adr_nightly` + `occupancy_pct`; same NOI/cap keys)
  - `fix_flip` → **`flip_v1`** (needs `arv`; `hold_months` default 6)
  - `wholesale` → **`wholesale_v1`** (needs `arv` + ask)
  - `redevelop` → **`redevelop_v1`** (needs `arv` as exit/land value; CapEx = demo/site; `hold_months` default 18) — see the **`redevelopment`** skill
- **NOI** = levered CF + P&I (tax/insurance/HOA/vacancy/maintenance + itemized `monthly_opex_total` are OpEx). Cap rate = annual NOI / purchase. Not used for primary-purpose PITI.
- **`deal_purpose=primary`** (owner-occupied): skips investment models; persist `raw.cashflow` as **`piti_v1`** (PITI / monthly housing cost) when ask/purchase is present; **keep `score`/`confidence` null** and **`income_strategy` null**. This is not an investment score. **Do not use `advance_deal`.** Matching uses the blob (or the deal’s set financing profile) for cash/rate gates. Do not map PITI to monthly cashflow. Flipping purpose back to investment/either restores `ltr` if strategy is still null.
- When `financing_profile_id` is null, scoring uses the strategy default: LTR/STR → `traditional_good`; flip/wholesale/redevelop → `cash_offer` (same as the UI's blank-profile default). Null-profile flips therefore read as all-cash theses in buyer matching.
- Default income strategy is `ltr` for investment / either. Primary has no strategy.
- **Expenses:** Acquisition CapEx (`repair_items` → `repair_estimate` → `cash_in`) via the **`cost-estimate`** skill; Recurring OpEx (`opex_items` → `monthly_opex_total`) via the **`opex`** skill. **Never** bump `maintenance_pct` / `insurance_annual` as a CapEx or OpEx proxy.

## Pipeline

```
ingest → research (+ trends if stale) → triage → scenarios → underwrite (this workflow)
```

This workflow **owns verdict + status**. For **score-ready investment deals** (no blocking `data_gaps`), prefer MCP **`advance_deal(dry_run=true)`** first — it composes trends → economic candidates across the scenario grid → **buyer-first cell pick** → proposed status. Review the proposal, then `advance_deal(dry_run=false)` to write. Keep discrete tools (`create_scenario_run`, `match_deal_buyers`, `update_deal_status`, …) for surgery and edge cases. Matrix scan / variations / select / apply details live in the **`scenarios`** skill. Evidence gathering lives in the **`research`** skill (which may load the **`trends`** skill).

**`advance_deal` stops** (no status write): blocking gaps; `deal_purpose=primary`; `auction_date` set (`needs_distress_review`); multi-geo with `primary_geography_level` still unset (normally auto-set when geos are linked). Defaults: `dry_run=true`, `link_matches=false`, `apply_scenario=false`.

## Aggressiveness

Defaults by `income_strategy` (see the **`realtypad`** skill). Chat override: `aggressiveness: aggressive|moderate|strict` (or “be aggressive” / “be conservative”) applies to the whole run. **`strict`** (“be conservative”): LTR/STR fallback CF ≳ **~$200/mo** (≈ Economics ~70) or a clearly stronger hold than moderate; flip ≳ **$50k**; wholesale ≳ **$40k**.

**Prefer projections over the month-0 hard bar whenever possible (ltr/str).** Run `cashflow_projections` before verdict. When the projection succeeds, rank/pass from the hold thesis (IRR, equity multiple, growth assumptions) — not from month-0 CF alone. A deal can clear the **economic** bar with thin/negative month-0 CF if the hold projection is solid under disclosed growth; conversely, fail a “meets $100/mo” deal if the hold path looks weak. Fall back to the CF hard bar only when projections skip/fail (not ltr/str, missing inputs, etc.).

| Strategy | Mode | Clears economic bar when (verified inputs) | Fail economics / deprioritize |
|----------|------|--------------------------------------------|-------------------------------|
| `ltr` / `str` | **moderate** | **Preferred:** solid hold projection (cite IRR + equity multiple + `assumptions_source`). **Fallback** if projection unavailable: CF ≳ **~$100/mo** (≈ Economics ~55–60). | Weak/negative hold thesis; or (fallback) thin/negative CF; capped rent unresolved; `repairs_unknown` (AS-IS CapEx unset); structural blockers |
| `fix_flip` | **aggressive** | Clear asymmetric profit (≳ **$40–50k** or Economics ≳ **~80**) with ARV/repairs evidence | Breakeven / thin margin |
| `wholesale` | **aggressive** | Clear asymmetric spread (≳ **$25–40k** or Economics ≳ **~70**) | Thin assignment spreads |
| `redevelop` | **aggressive** | Clear asymmetric residual (≳ **$40–50k**) with exit/land value evidence — not renovated flip ARV | Speculative unit mix / no as-is value |

LTR/STR language: prefer “solid hold / investable projection”; month-0 CF is secondary. Flip/wholesale: “asymmetric.” Speculative STR ADR/occ only → stay `researching`. Treat `repairs_unknown` (needs_rehab + null repair_estimate) like rent capped — stay `researching` until CapEx is estimated via **repair items** (or rehab flag cleared); do **not** invent repairs into OpEx or raise `maintenance_pct`. When `repair_estimate` is set, cite hold `cash_in` (down + CapEx) — CoC/IRR use that basis; monthly CF still excludes CapEx.

### Buyer-fit gate (required before `ranked`)

Economics alone is **not** enough to pursue. After a deal clears the economic bar (and is not a structural pattern-pass):

1. Prefer **`advance_deal`**: it scores the buyer book against **every economically viable grid cell**, then **selects** the best-Economics cell among those with ≥1 buyer fit (buyer-first). Manual path: persist pursue theses via the **`scenarios`** skill (`create_scenario_run` → **select**). You do **not** need to `apply` just to match — matching scores **live UW + every selected scenario snapshot** (multi-thesis). **`apply_scenario_run`** sets `deals.selected_scenario_run_id` (Overview / Buyers UI default); it does not narrow matching.
2. Call **`match_deal_buyers`** (default `include_snapshots=true`) when not using advance. Each match carries `scenario_run_id` / `scenario_run_label` (and `matrix_cell_key` when matching ephemeral grid cells).
3. **`match_count > 0`** → `ranked` (pursue). Prefer `link_matches=true` (auto-sets `deal_buyers.scenario_run_id`) or manual `link_deal_buyer(..., scenario_run_id=…)`. Name buyers **and theses** in the UW comment.
4. **`scanned > 0` and `match_count == 0`** → `passed` — cite “no buyer fit” plus strongest near-miss gaps (and which thesis was closest if useful).
5. **`scanned == 0`** (empty buyer book) → set **`watch`** (UI: Hold — no buyers yet). Research is done — keep an eye out for buyers; do **not** auto-`passed` and do **not** leave stuck in `researching`.

Near misses do **not** count as matches. Structural blockers still `passed` **before** this gate. Hard external/missing facts that block UW → **`blocked`**. **Distress override:** stay `researching`/`blocked` until tax/title/repairs are credible, then apply this gate — see the **`distress`** skill.

Overview **Theses** tabs default to the applied run (`selected_scenario_run_id`) when set; buyer-linked snapshots still appear. The Deal report lists **Buyers on this thesis**.

## Market geography for the underwriting worksheet

Deals often have a full geo chain (neighborhood / zip / city / county / MSA / state) with different ZHVI, ZORI, and growth paths. **Before** stuffing market numbers into UW (rent sanity vs ZORI, hold-growth / `cashflow_projections` / `get_market_projections`, appreciation context, “typical value” vs ask):

1. Ensure trends are fresh via the **`trends`** skill — geos via **`ensure_geo_markets`**, then **`refresh_deal_trends(gap_fill=true)`** when missing/stale.
2. Show a **short geo comparison** (latest sale index, YoY, median rent / YoY when present, `period_count`) from `deal.markets[]`.
3. **Ask the user which geography level to use** for worksheet / projection inputs (e.g. “Zip 78752 ★ vs St. Johns vs Travis County vs Austin MSA?”).
4. Only after they choose: pull `get_latest_market_snapshot` / `get_market_projections` / growth assumptions from **that** `market_id`, cite the level + name in the UW comment, and proceed with scenarios / `cashflow_projections`.

**Defaults if the user already named a layer** in chat (“use zip”, “MSA growth”) — honor that and skip the question. If only one linked market has usable series, state that you’re using it and continue. Hold CAGR defaults already use a composite of local geos (+ appraisal when usable); do not invent other blends for worksheet comps.

Changing the deal’s **primary** `market_id` / `primary_geography_level` can be done in the UI
(**Underwriting → Projection market**) or via `update_deal`. That selection is the **chart /
Prophet-path anchor** (and chips). **Hold CAGR defaults** use a **composite mean** of linked
local geos (`zip` / `city` / `county` with YoY) plus appraisal YoY when the series is usable —
not the primary market alone. Growth score stays its own multi-geo rollup (no appraisal blend).
`primary_geography_level` is auto-set when geos are linked if unset. The Trends tab is view-only
(finest-first; no primary star).

## Checklist

```
UW Progress:
- [ ] 1. Load deal (`get_deal` **core** by default) + `list_comments` / open `action_needed` via list tools or `view=overview` + `list_scenario_runs`
- [ ] 2. Confirm income_strategy
- [ ] 3. If needs_input / missing tax|HOA|rent|ADR|ARV|description|coords → the **`research`** skill
- [ ] 3c. STR (and LTR when relevant): itemize **Recurring OpEx** via the **`opex`** skill / `add_deal_opex_items` (cleaning/utilities/pool/PM/`capex_reserve`) — never bump `maintenance_pct`/`insurance_annual` as a blended proxy. Day-0 rehab → **Acquisition CapEx** via cost-estimate / repair items (not OpEx). Never duplicate the STR platform fee as an opex line. Optional starters: `get_uw_cost_defaults().opex_templates`.
- [ ] 3b. Trends: if multi-geo markets[] with series, present geo comparison for context. Hold CAGR defaults already composite local geos (+ appraisal when usable); ask the user only when overriding the **projection market** (chart / Prophet anchor) or worksheet comps. When county/msa/state supply/demand series exist (permits / jobs / pop / vacancy), cite them as **advisory context** in the UW comment — do not overwrite deal `vacancy_pct` or projection growth from them.
- [ ] 4. Confirm purchase assumption (ask | bid | offer)
- [ ] 5. Prefer **`advance_deal(dry_run=true)`** when score-ready; review proposal → `dry_run=false` to write. Else the **`scenarios`** skill: create/fork run → select cell → apply when promoting
- [ ] 5b. ltr/str: cashflow_projections (horizon default 5; required before verdict when possible) — optional scenario_run_id to project a cell without applying; cite IRR + assumptions_source + **chosen geo level**; prefer this over month-0 CF bar (advance_deal runs a hold check when it can)
- [ ] 5c. If economics clear and not using advance_deal: match_deal_buyers (snapshots on by default) — pursue only when match_count > 0; link with scenario_run_id; else passed (or watch if scanned=0)
- [ ] 6. Set status + **short Status note** (`rationale`, ≈2 sentences — one-line verdict / pattern; do **not** paste the UW comment body). Leave Data confidence bucket to rescore — skip if advance_deal already wrote status
- [ ] 7. `add_comment` (author=agent) with the **full** verdict + sources + scenario_run_id (+ IRR / hold thesis when projected) + **market geo used** + buyer match summary — skip if advance_deal already wrote
- [ ] 8. Brief user (table for multi-deal batches)
```

## MCP tools

RealtyPad MCP tools (same API as the hosted app):

| Step | Tool |
|------|------|
| Load | `get_deal` (core; `view=full` rare), `list_comments`, `list_deal_comps`, `list_scenario_runs`, `list_deal_opex_items` |
| **Pipeline (prefer)** | **`advance_deal`** (`dry_run=true` first) — gaps → economic grid candidates → buyer-first thesis → proposed/applied status |
| Money / profile / copy | `update_deal` (auto-rescored; money only — no status/rationale; prefer research workflow for bulk field fills); OpEx lines via `add_deal_opex_item(s)` |
| Scenario matrix | the **`scenarios`** skill / `create_scenario_run` → select → apply — **never** `cashflow_scenarios` (Preview-only dry-run) |
| Hold projection | `cashflow_projections` (ltr/str dry-run; growth: Prophet/OLS → YoY → 0%; optional `growth_schedule` for staged rates; optional `scenario_run_id`) |
| Market trajectory | `get_market_projections` (Prophet/OLS; after trends refresh; prefer `series=` + `include_projected=false`; check `fit_quality`) |
| Buyer fit | `match_deal_buyers` (scores live + selected snapshots; returns `scenario_run_id`) ; optional `link_matches=true` / `link_deal_buyer` |
| Force rescore | `rescore_deal` |
| Notes | `add_comment` (full UW brief); short Status note via **`update_deal_status`** `rationale` only; `add_observation` only for `action_needed` handoffs; `resolve_observation` when one is handled |
| Comps | `list_deal_comps` (research stores via scrapers + `add_deal_comps` / `add_deal_comp`) |
| Photos / files | `add_deal_attachment_urls` (gallery bulk) / `add_deal_attachment_url` |

## Hold growth defaults (expense / tax)

When `expense_growth_pct` is **unset**, `cashflow_projections` auto-selects expense growth from deal `appraisal_tax_history` (not CPI, not rent growth):

1. Tax Prophet (≥12 annual points) → `assumptions_source.expense_growth_pct = tax_prophet`
2. else tax OLS (≥3) → `tax_ols`
3. else tax YoY (last vs prior) → `tax_yoy`
4. else `0` / `zero`

That rate grows **tax + insurance + HOA** in simple mode (and insurance/HOA when a predictive tax path replaces tax levels). Override explicitly with `expense_growth_pct` or a staged `growth_schedule.expense` list (`len == horizon_years`; `rates[i]` produces year `i+1` from the prior level).

Parallel auto-defaults for rent/value: **composite mean** of linked local (`zip`/`city`/`county`) snapshot YoYs (value also averages in appraisal YoY when usable) → else `0`. Primary `market_id` is still used for chart / Prophet path fits. Always cite `assumptions_source` (e.g. `composite_local_appraisal`) in UW comments.

## Status decision tree

Use this before flipping status (UI Triage / deal Triage shows the same guide). Status code **`watch`** stays `watch` — label “Hold — no buyers yet.”

1. Hard external gap (title, auction terms, unverifiable tax)? → **`blocked`**
2. Gaps remain / economics not cleared? → **`researching`** (or leave **`new`** if untouched)
3. Below economic bar / structural reject? → **`passed`**
4. Economic candidates on the scenario grid → **`match_deal_buyers`** (advance matches all clearing cells; manual path select then match):
   - `scanned=0` → **`watch`** (keep an eye out for buyers — **not** more research, **not** `passed`)
   - `match_count≥1` → **`ranked`**
   - buyers exist, none fit → **`passed`**

**`watch` ≠ `passed` ≠ `blocked`.** Archive hides from queues without changing status. Human escalate `ranked` → `pursuing` → `closed`. Distress: stay `researching`/`blocked` until tax/title/repairs credible, then apply this tree.

## Status rubric

Apply the **Aggressiveness** table above (or the session override), then the **buyer-fit gate**. For ltr/str, decide economics from **hold projections first**; use the month-0 CF hard bar only as fallback.

| Outcome | Status | Typical Data confidence bucket |
|---------|--------|--------------------------------|
| Clears economic bar **and** `match_deal_buyers` returns ≥1 match | `ranked` | med–high |
| Economics clear but buyer book empty (`scanned=0`) — keep an eye out | `watch` | low–med |
| Clears economic bar but **no** buyer matches (`scanned>0`, `match_count=0`) | `passed` | med–high |
| Promising but gaps remain (auction, condition, unverified rent/ADR/ARV, projection skipped) | `researching` | low–med |
| Hard external fact missing (title, auction terms, unverifiable tax) | `blocked` | low–med |
| Below economic bar or structural blocker (weak hold thesis / thin CF fallback; MUD tax, HOA, affordability deed, thin flip/spread) | `passed` | med–high |
| Not yet touched | leave `new` (or `watch` if deliberately parked after research) | — |

Do **not** auto-archive during underwrite. Prefer `passed` for rejects; use `update_deal(archived=true)` only when the deal should leave default queues (passed clutter). Archive keeps status/UW/buyers and is excluded from `list_deals` unless `include_archived` / `archived_only`.

**Scores:** prefer MCP/API **`score_summary`**. Definitions (Economics / Data confidence / Growth / Drivers / Overall): [glossary](../realtypad/references/glossary.md). Bare `score` = Economics only — **never call Overall “score.”** **Early pass:** weak economics + negative overall/growth → lean `passed` unless distress or buyer override. Read `growth_factors` before Overall judgment. Fill data-confidence gaps before ranking keepers.

**high** Data confidence (LTR) when the checklist is ≥80 (typically real tax + insurance, rent not capped, repairs known if rehab). STR reaches high when tax/insurance verified **and** `income_verified=true` (confirmed ADR+occ — never comps medians alone); otherwise soft gap `income_estimated`. Flip/wholesale/redevelop need **ARV backed by ≥3 screened sale comps** (`raw.strategy.arv_supported`; 50% of the wholesale checklist, 40% of flip/redevelop) — a typed ARV with no comps cannot reach high; then set repairs/liens/hold and verified tax/ins. Adding or removing sale comps rescores the deal.

Verified-source definitions: see the **`research`** skill (do not duplicate fetch playbooks here).

## Wholesale / flip

Store ask/assignment, ARV + repair notes when known, spread/profit thesis, confidence `low|med|high`. Prefer model score when inputs exist. Derive flip/wholesale ARV from ≥3 sale comps ($/sqft → filtered comps → median last); the appraisal is an as-is value, never the ARV (redevelop: appraisal = exit/land value). See the research comps step; ≥5% method residual → explain or leave null. Use the **aggressive** economic bar — only continue to the buyer-fit gate when the spread/profit is clearly asymmetric. Do **not** present as legal/financial advice.

## Status note vs comment

- **`rationale` (Status note):** ≈2 sentences / ~280 chars when you set it — e.g. `Ranked — LTR @ traditional_good; IRR ~8%; 2 buyer matches.` First successful score may auto-seed a model one-liner when rationale is null; overwrite with a short verdict, not the UW essay. UI: header status chip → dialog; Home shows Status note + **data gaps** + comments (read story).
- **`comments`:** full research / UW briefs. UI: Overview.
- **`observations` (`action_needed`):** handoffs only. UI: Work tab.
- **`add_comment`:** owns the full template below. Do **not** paste the comment body into `rationale`.

## Comment template

```markdown
## UW — YYYY-MM-DD

**Verdict:** ranked | researching | watch | blocked | passed
**Strategy:** ltr | str | fix_flip | wholesale
**Scenario run:** `{run_id}` · selected `{strategy}` × `{profile_code}` · applied yes|no
**Purchase assumption:** $X (ask | bid | offer)
**Tax:** $Y/yr (source)
**HOA:** $Z/mo or none
**Income:** rent $R / ADR $A @ occ O% / ARV $V (source; capped?)
**Modeled:** ~$N (CF/mo | profit | spread) · score S · confidence C · model M
**Hold projection (ltr/str):** {horizon}y IRR ~X% · equity multiple Yx · growth rent=… value=… (assumptions_source) · **market geo:** {level} {name}
**Supply/demand context (if loaded):** permits / unemp / vacancy (source + geo) — advisory only
**Buyer fit:** scanned N · matches M · linked? yes|no — names / top gaps
**Blockers / gaps:** …
**Sources:** URL1, URL2
```

## Brief the user

- Single deal: short verdict + primary metric (ltr/str: lead with hold IRR / equity when projected) + **which market geo fed UW** + buyer-fit one-liner + one next action.
- Batch: a table (canvas / artifact view when the client has one) of address / verdict / corrected metric / buyer matches.
- If linked buyers have unread share-chat (`list_deal_buyers` → `unread_count`, or
  `list_buyer_message_threads`), follow the **`buyers`** skill —
  read the thread before recommending outreach; reply only when asked.

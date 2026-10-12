# Agent glossary

Shared terms for RealtyPad agents and humans. Load playbooks with the matching skill; use this page when a term is unclear.

## Deal status

| Status | Meaning |
|--------|---------|
| `new` | Ingest inbox — not yet active research |
| `researching` | Active WIP / soft gaps / still filling facts |
| `blocked` | Hard external or missing fact stops UW |
| `watch` | Economics clear, research done, **no buyers in the book yet** (`scanned=0`) — hold and watch |
| `ranked` | Economics clear **and** ≥1 buyer match |
| `passed` | Below economic bar, or buyers scanned but none fit, or structural pattern-pass |
| `pursuing` / `closed` | Human escalate after ranked |

**Decision tree:** hard gap → `blocked`; WIP/gaps → `researching`; below bar → `passed`; economics clear → `match_deal_buyers` → empty book `watch` | match ≥1 `ranked` | scanned none fit `passed`.

`watch` ≠ `passed` ≠ `blocked`. Archive is orthogonal.

## Gaps

| Term | Meaning |
|------|---------|
| `data_gaps` | Computed `{ code, blocking, detail }` on list/detail |
| **Blocking gap** | `blocking=true` — do not advance research → score/UW |
| **Soft gap** | Estimated or incomplete (e.g. `tax_estimated`, `missing_growth_drivers`) — research can continue with care. Soft for UW list filters; import/research automation still treats open `missing_growth_drivers` as ready-package pressure (pack eligible + critic done-gate). |

Filter helpers: `has_blocking_gap`, `has_soft_gap`, `has_gap=<code>`.

## Scores

Prefer MCP/API **`score_summary`**. Never call Overall “score.”

| Name | Range | What it is |
|------|-------|------------|
| **Economics** | 0–100 | CF / profit / spread quality (`deals.score` / `quality_score`) — triage `min_score` |
| **Data confidence** | 0–100 | Completeness checklist; `deals.confidence` enum is only the Low/Med/High bucket |
| **Growth** | −100…+100 | Strategy-aware market YoY (price ± rent/RevPAR blends) |
| **Drivers** | −100…+100 | Mean of active qualitative `growth_factors` (strength × direction) |
| **Overall** | −100…+100 | Blend of Economics (confidence-weighted) + Growth/Drivers — display hero (`total_score`) |

Bare `score` = Economics only. Early pass when weak economics + negative overall/growth.

## Money model

| Term | UI label | Role |
|------|----------|------|
| **Acquisition CapEx** | Fix-up costs | Repair items → auto `repair_estimate` → part of `cash_in` (not monthly CF) |
| **Recurring OpEx** | Monthly costs | OpEx items → `monthly_opex_total` (additive to LTR/STR OpEx) |
| `cash_in` | — | Down payment + (repair_estimate or 0) for CoC/IRR denominators |

Never bump `maintenance_pct` as a CapEx/OpEx proxy. CapEx playbook: the **`cost-estimate`** skill. OpEx / Monthly costs: the **`opex`** skill.

**Units on writes (validated):** deal `vacancy_pct` / `maintenance_pct` / `occupancy_pct` are **fractions 0–1** (5% → `0.05`); buyer `target_roi_pct` / `financing_rate_pct` are **percents** (8% → `8`). Money is ≥ 0 (ask / purchase / ARV / comp price > 0). Beds/baths 0–50, sqft 100–100,000, comp distance ≤ 100 mi. Dates (`as_of`, `event_at`, `list_started_at`, `captured_at`) must be between 1950 and tomorrow; `auction_date` up to 2 years out. Rent comps over $50k/mo or STR ADR over $10k/night fail as "looks like a sale price". A failed write names the field and usually the fix — correct and retry.

**URLs on writes (validated):** every `source_url` / listing URL must be a full `https://…` link (not "zillow", "N/A" or "see listing"); blank clears it. Photo fields (`image_url`, gallery `image_urls`, image attachments, contact `photo_url`) must be real photo URLs — no `data:` / `javascript:` / file paths, no Google Static Maps / Street View / satellite images, and not a listing *page* (open the page and use a gallery image URL). A bad gallery photo is reported in `photo_errors` and the rest still attach; a listing URL for another address is skipped and reported in `listing_url_errors`.

**Validation errors (every MCP tool):** a bad call returns `{"error": "validation_failed", "fields": [{"loc", "msg", "hint"?, "allowed"?}]}` instead of raising. `loc` names the argument or bulk row (`items[3].image_url`), `allowed` lists enum values, and `hint` gives the fix (`did you mean 'image_url'?`). Bulk tools reject unknown keys rather than silently dropping them. Observation `kind` is normalized (`Action_Needed` → `action_needed`); `state` must be a 2-letter USPS code; `postal_code` is `12345` or `12345-6789`. Fix the named fields and retry — do not retry the same call unchanged.

## Strategies and purpose

| Field | Role |
|-------|------|
| `deal_purpose` | `investment` (default) \| `primary` \| `either` — primary stores null `income_strategy` and uses PITI housing cost (no investment score) |
| `income_strategy` | Active thesis: `ltr` \| `str` \| `fix_flip` \| `wholesale` \| `redevelop` (null when primary) |
| `strategy_intent` | Optional focus list for research / gaps / UW fill; empty → fall back to primary strategy |

**Financing default by strategy:** LTR/STR → `traditional_good`; flip/wholesale/redevelop → `cash_offer`.

## Scenarios

| Term | Role |
|------|------|
| **Scenario run** | Durable UW scan (`create_scenario_run` → select → optional apply) — **only** decision path for theses |
| **Preview matrix** | Dry-run `cashflow_scenarios` — UI Preview only; never for verdicts or buyer theses |
| **Select** | Marks the pursue cell(s); matching scores live UW + selected snapshots |
| **Apply** | Promotes strategy/profile/overrides onto the deal; sets Overview/Buyers default thesis |

Prefer `advance_deal(dry_run=true)` on score-ready investment deals. Details: the **`scenarios`** skill.

## Buyer-fit gate

After economics clear (and not a structural pattern-pass):

1. Persist pursue theses (`create_scenario_run` → **select**; apply optional)
2. `match_deal_buyers` (live UW + selected scenario snapshots)
3. ≥1 match → `ranked` · scanned none fit → `passed` · empty book (`scanned=0`) → `watch`

Near misses do not count. Matching respects `buyer_type` vs `deal_purpose`. **Distress:** stay `researching`/`blocked` until tax/title/repairs are credible, then apply this gate.

## Narrative write map

Write each story once (Markdown in the UI):

| Layer | Job | Not for |
|-------|-----|---------|
| `listing_description` | Verbatim portal/MLS remarks | Agent analysis |
| `rationale` (**Status note**) | Short overwritten note (~2 sentences) | Full UW/research essays |
| Comment `body` | Append-only research/UW briefs, human chat | Structured comps/tax/CapEx rows |
| Observation (**Handoffs**) | `action_needed` next-agent todos only | Research diary / freeform notes |

When revisiting: comments + open handoffs + structured tabs — not a second freeform diary.

When a handoff's to-do is handled, mark it done with `resolve_observation(deal_id, observation_id)` (it leaves `open_handoff_count` and the open list; `resolved=false` reopens). Don't delete or rewrite it to close it. Automation handoffs that only list data gaps close themselves (`raw.resolved_reason=gaps_cleared`) once none of those gaps remain, on the next deal save/rescore or automation step; ones with a lead line (failed run, critic note) stay open until someone marks them done.

## Automation personas

Hosted Quick Analyze / goal automation seats (PDLC-style RACI). Full map + control loop: `read_doc("agents/automation-planner.md")`.

| Seat | Meaning |
|------|---------|
| **Goal steward** | Product/API sets `goal` / focus / aggressiveness — what “done” means. Not an LLM loop. |
| **Manager** | Budget & WIP rails (step/retry caps, paid-API / Actor allowlists). Code today; optional LLM later for marginal spend. |
| **Screener** | Early junk/non-fit kill before deep research *(not shipped)*. |
| **Planner** | Picks next eligible **pack** + free-text **directive** / `done` / `pause`. Does not grade its own work. |
| **Workers** | Per-pack ReAct + deterministic apply — produce artifacts; do not redefine the goal. Follow the planner directive. |
| **Critic** | Reviews each worker turn and gates `done` (`critic_N`) with **read-only MCP** to verify saved comps/tax/CapEx/OpEx; `accept`, `follow_up` (sent straight to the worker), or `pause`. Code rails on `done`: foreign listing URLs, disclosed repairs with fix-up costs unset (soft gap `disclosed_repairs_unset`), missing growth drivers. |
| **Brief** | End-of-run human accept package → `result_summary` (`brief_1`); must name open risks, not only cashflow. |
| **Research session** | How hosted automation runs: planner asks → worker (one continuous conversation, full toolset minus guardrails) → hooks → critic. See `read_doc("agents/automation-planner.md")`. Replaces the retired "pack" steps. |
| **Directive** | Free-text worker mission from the planner (shown as plan rationale; injected into the worker prompt). |

Separation of duties: planner ≠ critic; brief does not mutate money fields.

## Quick enums

| Field | Values |
|-------|--------|
| `build_type` | `resale` \| `new_build` (optional; not property type) |
| `property_type` | `single_family` \| `condo` \| `townhouse` \| `duplex` \| `triplex` \| `multi_family` \| `other` |
| `amenities` | Closed catalog on `properties.amenities` / buyer criteria: `pool` \| `hot_tub` \| `garage` \| `fireplace` \| `ac` \| `basement` \| `fenced_yard` \| `waterfront` \| `view` \| `elevator` \| `gym` \| `community_pool` |
| `attributes` | JSONB bag on `properties.attributes` for MLS-style facts beyond the amenity catalog. Soft convention keys (not an enum): `stories`, `flooring`, `cooling`, `heating`, `appliances`, `interior_features`, `security`, `exterior` `{roof,foundation,siding,features,lot_description,front_door_faces,private_pool}`, `parking` `{garage_spaces,spaces,type}`, `utilities`, `dwelling_type`, `community_features`, `subdivision`, `legal_description`, `mls_id`. Top-level shallow merge via `update_deal_property` (null deletes a key). Agents research portal detail pages (HAR, Zillow, …) and patch — **never invent**; **do not** build portal feature adapters. HOA/tax/insurance → `update_deal` money fields, not attributes. |
| `income_strategy` | `ltr` \| `str` \| `fix_flip` \| `wholesale` \| `redevelop` |
| `deal_purpose` | `investment` \| `primary` \| `either` |

## MCP tool tags

Client grouping tags on tools (`meta._fastmcp.tags`): `ingest`, `deals`, `research`, `triage`, `scenarios`, `underwrite`, `trends`, `financing`, `contacts`, `buyers`, `guides`. Prefer `search_docs` then the matching skill.

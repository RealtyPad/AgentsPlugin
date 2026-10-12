---
name: redevelopment
description: >-
  Underwrite RealtyPad land, tear-down and densification deals with the
  redevelop strategy: exit/land value, demo and construction CapEx, residual
  math, zoning checks. Use when a deal is a lot, tear-down, teardown_signals
  QA flag, ADU/lot-split/densification play, or the user says redevelop,
  redevelopment, land, or tear-down.
---

# Redevelopment / land play

Use when the thesis is **lot value, tear-down, densification, or multi-unit entitlement** — not a renovated SFR flip.

## When this applies

Detect from listing copy or human intent:

- “development opportunity”, “multi-unit”, “tear-down”, “lot value”, “buildable”, “HOME Initiative”, “infill”, “entitlement”
- Aerial-only photo packages of a lot with a tired shell
- Buyer book wants land / builders / densification

Set **`income_strategy=redevelop`** (or add `redevelop` to **`strategy_intent`**). Do **not** force `fix_flip` ARV from renovated comps. Wholesale remains valid for *assignment into* a redev buyer; redevelop is the *site thesis*.

Land lots without a house: use `property_type=other` plus `lot_size_acres` and zoning notes in `attributes` (no separate land enum in v1).

### Teardown / land-value listings: condition decides the strategies

When the listing says **tear down**, **lot / land value**, or **uninhabitable** (`qa_flags.teardown_signals`) and the photos agree, the existing house is not a rental or a flip:

- Narrow **`strategy_intent` to `[redevelop]`** (add `wholesale` only for an assign-to-builder thesis). Remove `ltr` / `str` / `fix_flip`.
- Do **not** fill rent, ADR, or a renovated flip ARV just to clear a blocking gap. If they were filled that way, clear them (`update_deal` with null).
- Pick one exit thesis and make the numbers match it:
  - **Land play** (sell or hold the lot): `arv` = **land value** — lot / teardown sale comps, or the **land portion** of the assessment. Not the total as-is appraisal, which still counts the house. Budget = demo + site.
  - **Build** (new construction): `arv` = **new-build value** from nearby new-construction sale comps. The budget must include **construction** as well as demo + site, priced as repair items from sourced local $/sqft ranges (the **`cost-estimate`** skill).
- If the photos show the house is actually usable, keep the other strategies and say why in a comment, citing the evidence.

## Research checklist

1. Confirm ask / purchase and lot size; capture listing write-up + gallery (even if aerial).
2. Sale comps: prefer **lot / tear-down / as-is / multi-unit** sales nearby — not pretty renovated 2/1 flips.
3. Appraisal / tax history → the **land** portion of the assessment is a land-value anchor when lot comps are thin. The total as-is value is not an exit for a teardown (it still counts the house).
4. Growth drivers: density policy, corridor projects, nearby mixed-use — qualitative go/no-go.
5. CapEx: **demo + site** (and soft costs if known) via repair items — not kitchen/bath BOM fiction. A **build** thesis adds construction, priced from sourced $/sqft ranges.
6. Zoning / entitlement: verify with the city when material; if unverified, note it and **pause** rather than invent unit counts.

## Underwriting (`redevelop_v1`)

| Field | Meaning |
|-------|---------|
| ask / purchase | Basis |
| `arv` | **Exit value**: land value (land play) or new-build value (build) — UI label, not flip after-repair |
| `repair_estimate` | Demo / site CapEx, plus construction for a build thesis (default 0) |
| `hold_months` | Default **18** |

Residual = exit − purchase − CapEx − hold carrying − interest carry − selling costs (7% of exit, `raw.selling_costs`) − buy closing costs (2% of purchase, `raw.buy_closing_costs`). Economic bar is **aggressive** / asymmetric (same order as flip ≈ $40–50k). Financing default `cash_offer`.

ARV method order still applies in hosted fill, but for redevelop an appraisal-based exit vs renovated $/sqft residual is allowlisted as `redevelopment_as_is`. If methods disagree ≥5% without an allowlisted reason, **leave `arv` null** and stop retrying — do not invent `residual_reason` via MCP.

## Status / buyers

Same pipeline as other investment strategies: clear economics → `match_deal_buyers` → `ranked` / `watch` / `passed`. Hard zoning gaps → `blocked` or stay `researching` with an `action_needed` handoff.

## Do not

- Treat assessed value as renovated flip ARV without a redevelop thesis
- Widen renovated SFR comps forever to “fix” residual
- Pair a new-build exit with a demo-only budget, or use the as-is appraisal as a teardown's exit
- Fill rent / ADR / flip ARV on a teardown just to clear a data gap
- Invent unit mix, construction loans, or JV waterfalls (out of scope for v1); construction costs need a cited source

---
name: scenarios
description: >-
  Persist and manage RealtyPad underwriting scenario runs: scan financing ×
  strategy matrices, fork bid/rent variations, select a cell, and apply it to
  a deal. Use when the user asks for a scenario scan, sensitivity analysis,
  bid variation, save scenarios, re-run scenarios after input changes, or when
  underwrite/triage needs a durable scenario decision artifact.
---

# RealtyPad — scenarios (decision artifacts)

## Hard rules

- **Never invent numbers.** Prefer null over fiction. Mark estimates in comments / `raw`.
- **Do not** auto-persist every rescore — only create/fork runs when scanning intentionally.
- Read existing runs (`scenario_get`) before forking a near-duplicate.
- **Do not** set triage verdict (`ranked` / `passed`) — that is the **`underwrite`** skill / the **`triage`** skill.
- Apply only when promoting a decision onto the deal; exploratory scans stay `draft` / unapplied.

## Scenario surfaces (write once)

| Surface | Job | Agents / UI |
|---------|-----|-------------|
| `raw.scenario_metrics` | Slim focus×profile rescore cache | Feeds list/detail strategy chips via `scenario_metrics` / `intent_score_summary` — **ignore for UW decisions** |
| `scenario_run(dry_run=true)` | Form-aware matrix dry-run (saves nothing) | **Preview only** — never UW verdict / buyer thesis |
| `scenario_runs` | Durable scan + select/apply | **Only** decision path; applied run = `deals.selected_scenario_run_id` |

UI **Preview** tab calls dry-run with form overrides — that is **not** the rescore cache. Overview / Buyers default to the applied run when set. `buyer_match` still scores live UW + every run with a selected cell (multi-thesis); apply is optional for matching.

## Owns vs does not

| Owns | Does not own |
|------|----------------|
| Create / fork / select / apply scenario runs | Tax/HOA/rent evidence (the **`research`** skill) |
| Matrix interpretation for sensitivity | Status rubric / aggressiveness |
| Bid / purchase / rent override variations | Pattern-pass clusters (triage) |
| Linking UW comments to `scenario_run_id` | Dry-run-only `scenario_run(dry_run=true)` (Preview only) |
| **Selected cells as buyer-match theses** (no apply required) | Buyer messaging (the **`buyers`** skill) |

## Pipeline

```
research → scenarios (this workflow) → underwrite (verdict + status + buyer_match)
```

Underwrite **delegates** matrix work here. Scenario-only passes (sensitivity, bid grid) stop after select or a brief — no status flip unless the user asks for full UW.

**Buyer matching:** `scenario_select` creates a durable thesis that `buyer_match` scores without applying onto the deal. **`deal_advance`** is buyer-first: it matches economically viable grid cells against the buyer book, then selects the best-Economics matching cell (and may freeze alternate draft theses when linking buyers whose best cell differs). Manual path: prefer selecting (and labeling) pursue bids before the buyer-fit gate; `link_matches` stores `deal_buyers.scenario_run_id`. Apply only when promoting the workspace live UW / Overview default.

## Checklist

```
Scenario Progress:
- [ ] 1. Load deal (`deal_get` core) + scenario_get; `comment_list` if revisiting
- [ ] 2. If needs_input / missing money fields → the **`research`** skill first
- [ ] 3. `scenario_run(deal_id, runs=[{label: "baseline ask"}])` — or set `fork_from_run_id` on a run to start from an existing one
- [ ] 4. Review top ok cells (score / CF / spread by strategy × profile)
- [ ] 5. Optional: more objects in `runs` (or a fork) with purchase_price / rent / ADR overrides; several variations go in one call
- [ ] 6. `scenario_select(deal_id, run_id, income_strategy, financing_profile_id)`
- [ ] 7. `scenario_apply(deal_id, run_id)` only when promoting settings onto the deal
- [ ] 8. comment_create with scenario_run_id + selected strategy/profile (if UW or user wants a note)
```

## MCP tools

RealtyPad MCP tools (REST under `/api/deals/{id}/scenario-runs` on the hosted API):

| Step | Tool |
|------|------|
| Load | `deal_get` (core), `scenario_get` (`run_id` optional: without it, the deal's runs) |
| Scan | `scenario_run(deal_id, runs[])`: each run takes optional money overrides + `strategies` (max 20 runs per call) |
| Variation | `scenario_run` with `fork_from_run_id` (merges overrides onto the parent) |
| Choose | `scenario_select` |
| Promote | `scenario_apply` → patches strategy + profile + overrides, rescored |
| Preview only | `scenario_run(dry_run=true)` — computes the matrix, writes nothing; **never** for UW verdict or buyer thesis |

### Apply surface

`scenario_apply` writes:

- `income_strategy` + `financing_profile_id` from the selection
- run `overrides` money/build fields (e.g. `purchase_price`)
- `deals.selected_scenario_run_id`

It does **not** invent fields that were not overridden. It does **not** rewrite live CapEx/OpEx line tables from the snapshot.

### CapEx / OpEx freeze

`scenario_run` (and share auto-freeze) copies into `base_inputs`:

- full CapEx BOM (`repair_items` + lines)
- full OpEx sheet (`opex_items` with `monthly_equiv`)
- calc breakdown (`opex_breakdown`) plus scalars `repair_estimate` / `monthly_opex_total`

Personalized buyer shares overlay those frozen sheets so Expenses never drift from the thesis after live edits. Workspace Expenses tab stays live (edit surface). Fork copies parent `base_inputs`.

## When not to apply

- Exploring “what if” grids for the user without changing the deal
- Inputs still needs-input / unverified and you are only mapping sensitivity
- Multiple variations still under discussion — leave them `draft` / selected-but-unapplied

## Comment snippet (optional)

```markdown
**Scenario run:** `{run_id}` ({label})
**Selected:** `{income_strategy}` × `{profile_code}`
**Overrides:** purchase=$X (or none)
**Applied:** yes | no
```

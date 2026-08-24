---
name: scenarios
description: >-
  Persist and manage RealtyPad underwriting scenario runs: scan financing ×
  strategy matrices, fork bid/rent variations, select a cell, and apply it to a
  deal. Use when the user asks for a scenario scan, sensitivity analysis, bid
  variation, save scenarios, re-run scenarios after input changes, or when
  underwrite/triage needs a durable scenario decision artifact.
---

# RealtyPad — scenarios

Call `get_agent_manual(workflow="scenarios")` before running this workflow. Follow that Markdown.

## Hard rules

- Durable path = `scenario_runs` (create → select → optional apply). `cashflow_scenarios` is **Preview-only** dry-run — never UW verdict or buyer thesis.
- Research must have already filled verified inputs. Do not invent rent, tax, ADR, or ARV to force a matrix.
- Select creates a matchable thesis without apply. Apply sets live UW / Overview default (`selected_scenario_run_id`).
- Runs freeze CapEx/OpEx sheets into `base_inputs` for buyer shares. Underwrite owns the verdict.

## Typical tools

`create_scenario_run`, `list_scenario_runs`, `get_scenario_run`, `fork_scenario_run`, `select_scenario_run`, `apply_scenario_run`, `cashflow_projections`. Use `cashflow_scenarios` only for Preview dry-run.

## Checklist

```
Scenarios Progress:
- [ ] Confirm deal inputs are complete enough to score
- [ ] create_scenario_run (financing × strategy matrix)
- [ ] Fork bid/rent/repair variations if asked
- [ ] select_scenario_run for the chosen cell (thesis for match_deal_buyers)
- [ ] apply_scenario_run only when promoting onto the deal
```

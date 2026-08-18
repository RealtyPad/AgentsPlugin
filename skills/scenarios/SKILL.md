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

- Scenarios are decision artifacts. Do not treat a one-off `cashflow_scenarios` printout as the saved run.
- Research must have already filled verified inputs. Do not invent rent, tax, ADR, or ARV to force a matrix.
- Underwrite owns the verdict. This skill persists the matrix, forks, selection, and apply.

## Typical tools

`cashflow_scenarios`, `cashflow_projections`, `create_scenario_run`, `list_scenario_runs`, `get_scenario_run`, `fork_scenario_run`, `select_scenario_run`, `apply_scenario_run`.

## Checklist

```
Scenarios Progress:
- [ ] Confirm deal inputs are complete enough to score
- [ ] create_scenario_run (financing × strategy matrix)
- [ ] Fork bid/rent/repair variations if asked
- [ ] select_scenario_run for the chosen cell
- [ ] apply_scenario_run only when the user wants it on the deal
```

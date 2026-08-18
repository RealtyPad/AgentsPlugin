---
name: triage
description: >-
  Process the RealtyPad triage queue in batches: needs-input, shortlist, or
  regional cluster passes. Load deals via list_deals, refresh stale market
  trends for the cluster, call research for data gaps, pattern-pass or
  underwrite, update statuses, and brief. Use when the user asks to triage,
  clear the queue, needs input, shortlist, regional pass, or batch-pass a
  market/builder cluster.
---

# RealtyPad — triage

Call `get_agent_manual(workflow="triage")` before running this workflow. Follow that Markdown.

## Hard rules

- Prefer evidence over score. Scores with estimated tax or capped rent lie.
- Fill data gaps with the `research` skill before judging.
- Regional/cluster passes: refresh stale markets once via `trends`.
- Individual verdicts: `underwrite` (which calls `scenarios`). Pattern-pass only when the failure mode is already documented.
- Read comments before flipping a deal that was previously researched.

## Queue modes

Match UI `/triage` (`MIN_SCORE = 60`):

- **needs_input** (default): `list_deals` with `needs_input=true` on `researching` and on `watch` with `min_score=60`.
- **shortlist**: high-score researching/watch without blocking input gaps.
- **cluster**: same market/builder/pattern; refresh trends once, then batch.

## Aggressiveness

`ltr`/`str` moderate (prefer hold projections; fallback ~$100/mo). `fix_flip`/`wholesale` aggressive (asymmetric only). `strict` raises floors. Pursue (`ranked`) only after `match_deal_buyers` finds ≥1 buyer fit, except structural pattern-passes.

---
name: triage
description: >-
  Process the RealtyPad triage queue in batches: new inbox, Blocking gaps,
  Soft gaps, blocked/watch holds, ranked/pursuing stages, or regional cluster
  passes. Load deals via list_deals, refresh stale market trends for the
  cluster, run research for data gaps, pattern-pass or underwrite, update
  statuses, and brief. Use when the user asks to triage, clear the queue, new
  inbox, blocking gaps, soft gaps, needs research, needs verification,
  blocked, regional pass, or batch-pass a market/builder cluster.
---

# RealtyPad — triage

## Hard rules

- Prefer **evidence** over Economics alone — Economics with estimated tax or capped rent lie.
- **Scores:** prefer `score_summary`. Economics / Data confidence / Growth / Overall — see [glossary](../realtypad/references/glossary.md). Never call Overall “score.” `min_score` filters Economics only.
- Read **`score_summary`** and **`data_gaps`** on `list_deals` / `get_deal` (**core**) first.
- **`data_gaps`**: list of `{ code, blocking, detail }`. Any `blocking: true` gap means do **not** advance research → score/UW / claim ready. Soft gaps (`tax_estimated`, `missing_growth_drivers`, etc.) can proceed with eyes open. Codes: `photos`, `rent`, `adr`, `arv`, `unscored` (investment / either only), `housing_cost_missing` (primary only — no ask/purchase or no `piti_v1` housing cost; primary deals keep `score` null by design), `tax_implausible` (blocking: stated `tax_annual` under $100/yr or under 0.1% of purchase — a placeholder like $1; scoring uses the workspace tax estimate until a real tax bill is entered), `tax_estimated`, `insurance_estimated`, `vacancy_estimated`, `hoa_estimated` (insurance/vacancy/HOA still equal to an automation market-trend fill — cleared when a verified value replaces it), `rent_capped`, `repairs_unknown`, `missing_growth_drivers`. Strategy money gaps (`rent`/`adr`/`arv`) cover **`strategy_intent`** when set (union), else the single `income_strategy`.
- **Early pass (trash):** when Economics is already weak (≲ ~40 / fails the strategy economic bar on known inputs) **and** Overall is negative **or** Growth is clearly negative (≲ −30) with no distress/buyer override — set `passed` without a full research deep-dive. Still skim listing + comments; do not invent numbers to “save” it.
- **Prioritize research** on deals with blocking `data_gaps`, decent Economics but low Data confidence (`confidence_gaps` / soft gaps), or missing Growth income series (`growth_gaps`). Prefer `has_blocking_gap=true` / `has_gap=<code>`. Soft gaps queue: `has_soft_gap=true` (deprecated `needs_input` still aliases).
- Fill data gaps with the **`research`** skill before judging keepers.
- For regional/cluster passes, refresh stale markets once via the **`trends`** skill (dedupe deals; **`ensure_geo_markets`** + **`refresh_deal_trends(gap_fill=true)`** per deal when needed — not agent CSV → upsert; surgical refresh only when needed).
- For verdict + status on individual deals, follow the **`underwrite`** skill (which calls the **`scenarios`** skill).
- For scenario-only / sensitivity / bid variations without a full verdict, follow the **`scenarios`** skill.
- Never invent numbers. Pattern-pass only when the failure mode is already documented.
- Read **comments + open `action_needed` handoffs** before flipping a deal that was previously researched.

## Aggressiveness

Defaults by `income_strategy` (see the **`realtypad`** skill / the **`underwrite`** skill). Chat override: `aggressiveness: aggressive|moderate|strict` (or “be aggressive” / “be conservative”). **`strict`** (“be conservative”): LTR/STR fallback CF ≳ **~$200/mo** (≈ Economics ~70) or a clearly stronger hold than moderate; flip ≳ **$50k**; wholesale ≳ **$40k**.

| Strategy | Mode | Clears economic bar |
|----------|------|---------------------|
| `ltr` / `str` | **moderate** | **Prefer** hold projection (`cashflow_projections` via underwrite) over month-0 CF; fallback CF ≳ **~$100/mo** only if projection unavailable |
| `fix_flip` / `wholesale` | **aggressive** | Clear asymmetric profit/spread only (flip ≳ ~$40–50k / Economics ~80; wholesale ≳ ~$25–40k / Economics ~70) |

**Pursue vs pass:** economic bar is necessary but not sufficient. After a deal clears economics (and is not a structural pattern-pass), underwrite runs **`match_deal_buyers`** (live UW + selected scenario snapshots): matches → `ranked`; buyers scanned but none fit → `passed`; empty buyer book (`scanned=0`) → **`watch`** (Hold — no buyers yet; research done — keep an eye out). Hard external gaps → **`blocked`**. Human escalate `ranked` → `pursuing`. Full tree: the **`underwrite`** skill Status decision tree.

## Queue modes

Match UI `/triage` stage tabs (labels: New, **Blocking gaps**, **Soft gaps**, …):

### new (default inbox)

```
list_deals(status="new", limit=80)
```

Ingested, not started. Start research → `researching`, or pass / block.

### Blocking gaps (UI mode id `needs_research`)

Any blocking `data_gaps` entry (photos / rent / ADR / ARV / unscored / housing cost):

```
list_deals(status="new,researching", has_blocking_gap=true, limit=80)
list_deals(has_gap="photos", limit=80)
```

Use `has_blocking_gap=true` (same predicate as Blocking gaps).

### Soft gaps (UI mode id `needs_verification`)

Estimated tax/insurance, rent capped, `repairs_unknown`, listing off-market/auction, or missing listing contact/description:

```
list_deals(has_gap="tax_estimated", limit=80)
list_deals(has_soft_gap=true, limit=80)
```

`has_soft_gap` is the soft-gap OR convenience for this queue; prefer per-code `has_gap` when targeting one soft flag. Deprecated `needs_input` aliases `has_soft_gap` when unset.

### blocked / watch / ranked / pursuing

```
list_deals(status="blocked", limit=80)
list_deals(status="watch", limit=80)
list_deals(status="ranked", limit=80)
list_deals(status="pursuing", limit=80)
```

- **`blocked`**: hard external stop — unblock when the fact clears.
- **`watch`** (UI: Hold — no buyers yet): economics already clear; empty buyer book or parked — **not** more research; resume when buyers appear. New/reopened/changed buyers auto-flag fitting Hold deals with an `action_needed` handoff (`raw.tag=watch_resurface`) — work those first: `match_deal_buyers` → `ranked` (see the **`buyers`** skill).
- **`ranked` / `pursuing`**: pursue pipeline.

Hide low data-confidence `watch` unless the user asks to show them (UI default).

### regional / cluster

Filter by `q`, `market_id`, or shared builder/HOA/subdivision after loading a stage slice. Process the **cluster** as one thesis, not 50 isolated vibes.

## Batch workflow

```
Triage Progress:
- [ ] 1. Pick mode + scope (N deals or geography/builder)
- [ ] 2. Load queue via list_deals; sample before full pass; read data_gaps on each item
- [ ] 3. Regional/cluster: dedupe market_ids → the **`trends`** skill for stale (~35d) or below the 120-month target (unless source is known shorter) home + LTR rent (STR if str deals; request up to 10y)
- [ ] 4. Detect pattern (see below) vs needs research / full UW
- [ ] 5a. Pattern-pass: `update_deal_status` (single) or `update_deals` **status+rationale only** (cluster — no money in the same call)
- [ ] 5b. Keepers with blocking data_gaps → the **`research`** skill, then UW when clear
- [ ] 5c. Otherwise the **`underwrite`** skill (scenarios + buyer match)
- [ ] 6. Brief with a table: counts, verdicts, remaining queue
```

## Pattern-pass

When a whole cluster fails for the same documented reason (HOA tax bomb, builder premium, flood zone, etc.), batch `passed` with a shared short Status note (`rationale`) via **status-only** `update_deals` — do not invent per-deal numbers and do not mix money fields in that call. Single deal → `update_deal_status`.

## MCP

| Action | Tool |
|--------|------|
| Queue | `list_deals` (`has_blocking_gap`, `has_gap`, Soft gaps via `has_soft_gap`, `min_score`, `status`, `q`, `market_id`; read `score_summary` + `data_gaps`; archived excluded by default) |
| Detail | `get_deal` core (`score_summary` + `data_gaps` + counts); `view=overview`/`full` or list_* for nests |
| Status / Status note | `update_deal_status` (single); **`update_deals`** status+rationale **only** for cluster pattern-pass |
| Money / archive | `update_deal` / **`update_deals`** (money-only; `archived=true/false`) — never mix with status |
| Buyer fit (via UW) | `match_deal_buyers` (snapshots on by default; returns `scenario_run_id`) |
| Comment | `add_comment` |

## Brief format

Use a table (or the client's canvas / artifact view when available) for batch results:

| Address | Prior → verdict | Corrected CF | Buyers | Notes |
|---------|-----------------|--------------|--------|-------|
| … | new → passed | −$120/mo | — | tax $X |
| … | researching → ranked | +$180/mo | 2 matches | linked Diego |

Also give: how many cleared, how many left in queue, suggested **next cluster**.

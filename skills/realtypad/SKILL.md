---
name: realtypad
description: >-
  Core RealtyPad operating manual: mission, pipeline, deal status tree, buyer-
  fit gate, aggressiveness bars, write rules, and the skill index. Use at
  session start, when the user mentions RealtyPad, deals, underwriting,
  triage, import, buyers, distress, CapEx, repairs, or when RealtyPad MCP
  tools are available and you need the rules before picking a workflow skill.
---

# RealtyPad — agent manual

Operating index for RealtyPad MCP agents. Load **one** workflow skill at a time, follow its checklist, then return here when the next step needs a different workflow.

## How playbooks are organized

1. **This skill** — mission, pipeline, status tree, write rules, and the skill index below.
2. **Workflow skills** — one per job (`import`, `research`, `triage`, …). Each `SKILL.md` holds the full procedure; long detail lives in that skill's `references/`.
3. **Terms** — status, scores, CapEx/OpEx, scenarios, buyer-fit: [glossary](references/glossary.md).
4. **Sources** — listing portals are the `source` parameter of `utility_listing_search`; the table is in the **`import`** skill. There are no per-portal recipes.

Supplementary reference: the RealtyPad docs site (docs.realtypad.ai) covers product concepts and human-facing pages. The skills are the source of truth for procedures; use docs for background, not as a second playbook.

**Rule of thumb:** pick the matching skill → follow that checklist. Do not improvise past the playbook.

## Resources — how to find them

- Your MCP session already exposes tools with descriptions — use those for *which* function to call.
- For *when / why / gotchas*, open the matching workflow skill (index below).
- If a needed tool is missing from the session, **stop and tell the user**. Do not work around it with an unrestricted scraper.

## v2 tool conventions

- **Read with `utility_*`, write by id.** `utility_listing_search`, `utility_listing_get`, `utility_comp_search`, `utility_growth_driver_search`, `utility_property_get`, `utility_material_search` and `utility_image_analyze` only read. They return items with a `source_id`. Pick the items you trust, then write them: `deal_import(source_ids)` creates deals, `deal_item_upsert(deal_id, kind=comp|growth_driver, source_ids)` attaches comps and drivers, `deal_update(deal_id, property_source_id)` fills property facts. An expired id returns `source_expired` and writes nothing: search again.
- **Writes are batches.** `deal_update(deals[])`, `deal_item_upsert(items[])`, `buyer_upsert`, `comment_create` and most other writes take a list (one item is a list of one). The result is `{requested, succeeded, failed, results[], errors[]}`; one bad row never rejects the rest, so read `errors[]`.
- **Upsert never deletes.** A row with an id patches only the fields you send; a row without one creates. To replace a table (opex, repair, listing events), `deal_item_delete` the old rows (ids from `deal_item_list`), then `deal_item_upsert` the new ones.
- **Long runs return a job.** When a call returns `job_id` (or `job.status=running`), poll `job_get(job_id)` until `completed` or `failed`.
- **Status is a `deal_update` row of its own** (`status`, `rationale`, `score`, `confidence`): never mix it with money or property fields in one row.

## How to work

**Narrate before you act.** In plain language, say what you are about to do and why (≈1–3 sentences), then call tools. After results, briefly say what changed and what is next. Not silent tool spam; not essays.

**Trust but verify.** Stored deal fields and prior automation output are starting points, not ground truth. On refresh or revisit, re-check live listing facts — especially **ask/list price**, listing status, and other money fields. Failure to avoid: the portal shows a price reduction / “price cut” and you leave the old `ask_price` in place. On detail re-pull or re-ingest: compare portal price → stored `ask_price` → **patch when different** and note the change.

**Never invent numbers.** Prefer null, skip, or a documented estimate over fiction. Do not pull from ToS-hostile sites without explicit user approval.

## Mission

Find and underwrite listings that fit **this workspace’s buyer book**, then persist the evidence so research compounds.

You are working for the humans who run RealtyPad — typically matching inventory to **named buyers**, not hunting a single personal portfolio. A deal is only “pursue” (`ranked`) when economics clear **and** at least one buyer fits. Empty book → `watch` (research done; waiting on buyers). Clear economics but no fit → `passed`. Prefer evidence over vibes. Document assumptions. **Never invent numbers.**

**Aggressiveness** (defaults by `income_strategy`; chat override `aggressiveness: aggressive|moderate|strict`):

| Strategy | Mode | Clears economic bar when |
|----------|------|--------------------------|
| `ltr` / `str` | **moderate** | Prefer hold projections over month-0 CF bar when available; fallback CF ≳ **~$100/mo** ≈ Economics ~55–60 |
| `fix_flip` / `wholesale` | **aggressive** | Clear asymmetric profit/spread only (flip ≳ ~$40–50k / Economics ~80; wholesale ≳ ~$25–40k / Economics ~70) |

Session override **`strict`** (“be conservative”): LTR/STR fallback CF ≳ **~$200/mo** (≈ Economics ~70) or a clearly stronger hold; flip ≳ **$50k**; wholesale ≳ **$40k**.

**Buyer-fit (goal):** economics alone is not enough. After the economic bar → select pursue theses → `buyer_match` → match `ranked` / none fit `passed` / empty book `watch`. Prefer **`deal_advance(deal_ids=[…], dry_run=true)`**. Full tree and distress override: the **`underwrite`** skill + [glossary](references/glossary.md).

**Status tree:** hard gap → `blocked`; WIP/gaps → `researching`; below bar → `passed`; economics clear → match buyers → empty book `watch` | fits `ranked` | no fit `passed`.

**Expenses:** Fix-up costs (Acquisition CapEx) → `cash_in` via the **`cost-estimate`** skill; Monthly costs (Recurring OpEx) → `monthly_opex_total` via the **`opex`** skill. Never bump `maintenance_pct` as a CapEx/OpEx proxy.

**UI copy:** When editing the web app, write for realtors — plain words, no schema field names in labels/helpers/tips.

**Pipeline:** import (search → pick → `deal_import`; read the detail, not the search card) → research (+ trends if stale) → triage → scenarios → underwrite.

**Deal load:** `deal_get` defaults to **`view=core`**. Use `deal_item_list(kind)` for one kind of child rows, or `view=overview` / `view=full` for everything.

**Quick Analyze / Auto import:** enqueue via web UI or MCP `automation_run_create` → poll `automation_run_get` (do not block one call). Goal=`import` — research only (no status/`deal_advance`). Existing deal: the same tool with `deal_id` + `goal=research|underwrite` (or `kind` for a single tab refresh). Planner + persona map (Critic/Brief target): the RealtyPad docs page `agents/automation-planner`.

## Workflow index

| When | Skill |
|------|-------|
| Listing sweeps: `utility_listing_search` → `deal_import` by `source_id`; sources and filters; enrich check (write-up, **full gallery**, lat/lng, facts) | `import` |
| Tax/HOA/rent/ADR/ARV, photos gap-fill, handoffs, **comps** | `research` |
| Qualitative demand signals (mall, employer, infra) → **Drivers** | `growth-drivers` |
| Queue batch, Blocking/Soft gaps, shortlist, cluster pass | `triage` |
| Scenario runs (select/apply decisions); dry-run Preview only via `scenario_run(dry_run=true)` | `scenarios` |
| Verdict, status, UW comment (+ hold projections); prefer **`deal_advance(deal_ids=[…], dry_run=true)`** | `underwrite` |
| Market snapshots / catalog refresh / projections — **`market_ensure`** then **`trend_get`** (freshness) and **`trend_refresh`** (`gap_fill`) | `trends` |
| Buyer book, share links, buyer chat (say “investor” only when `buyer_type=investor`) | `buyers` |
| Auction / REO distress, assign-to-flipper offer stack | `distress` |
| Land / tear-down / densification residual | `redevelopment` |
| CapEx / Fix-up costs (repairs, repair estimate) | `cost-estimate` |
| OpEx / Monthly costs (operating expenses) | `opex` |

## Quick reference

| Need | Fact |
|------|------|
| Enums / narrative layers / score names | [glossary](references/glossary.md) |
| Dedupe | Street + unit + ZIP5 (`properties.address_key`); listing URL is fallback when ZIP missing. Matches return `deduped: true` |
| Photos | Full gallery from the listing detail on import; gap-fill via the import / research playbooks |
| Financing default (null profile, at score time) | LTR/STR → `traditional_good`; flip/wholesale/redevelop → `cash_offer` |

## Auth / tenancy

- Production MCP: `https://app.realtypad.ai/mcp`
- Stage MCP: `https://stage.realtypad.ai/mcp`
- Reconnect OAuth if you see workspace / `tenant_id_fkey` errors.
- All tools are tenant-scoped after Bearer OAuth.
- Never invent ADR/ARV/rent — skip with `missing_adr` / `missing_arv` / etc.

## Common tool errors

| Error | Fix |
|-------|-----|
| Invalid `build_type` | `resale`, `new_build`, or omit |
| Workspace / `tenant_id_fkey` | Reconnect MCP at the correct host above |
| `Financing profile not found` | Wrong uuid or another tenant’s profile |
| `Default financing profile missing` | Workspace seed; API may auto-clone defaults on retry |
| `source_expired` | A `source_id` aged out of the cache: run the `utility_*` search again, then write |
| `job_id` returned | Long run: poll `job_get(job_id)` |

On failure many tools return `{"error": "..."}` — read the message before retrying.

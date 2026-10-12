---
name: realtypad
description: >-
  Core RealtyPad operating manual: mission, pipeline, deal status tree, buyer-
  fit gate, aggressiveness bars, write rules, and the skill index. Use at
  session start, when the user mentions RealtyPad, deals, underwriting,
  triage, ingest, buyers, distress, CapEx, repairs, or when RealtyPad MCP
  tools are available and you need the rules before picking a workflow skill.
---

# RealtyPad — agent manual

Operating index for RealtyPad MCP agents. Load **one** workflow skill at a time, follow its checklist, then return here when the next step needs a different workflow.

## How playbooks are organized

1. **This skill** — mission, pipeline, status tree, write rules, and the skill index below.
2. **Workflow skills** — one per job (`ingest`, `research`, `triage`, …). Each `SKILL.md` holds the full procedure; long detail lives in that skill's `references/`.
3. **Terms** — status, scores, CapEx/OpEx, scenarios, buyer-fit: [glossary](references/glossary.md).
4. **Portal connector recipes** — [data-source catalog](../ingest/references/data-sources.md), then `../ingest/references/data-sources/<portal>.md`.

Supplementary reference: the RealtyPad docs MCP (`search_docs` with `search_type=keyword`, `read_doc`, `list_docs`) covers product concepts and human-facing pages. The skills are the source of truth for procedures; use docs for background, not as a second playbook.

**Rule of thumb:** pick the matching skill → follow that checklist. Do not improvise past the playbook.

## Resources — how to find them

- Your MCP session already exposes tools with descriptions — use those for *which* function to call.
- For *when / why / gotchas*, open the matching workflow skill (index below).
- If a needed connector or tool is missing from the session, **stop and tell the user** — do not use unrestricted Actors.

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

**Buyer-fit (goal):** economics alone is not enough. After the economic bar → select pursue theses → `match_deal_buyers` → match `ranked` / none fit `passed` / empty book `watch`. Prefer **`advance_deal(dry_run=true)`**. Full tree and distress override: the **`underwrite`** skill + [glossary](references/glossary.md).

**Status tree:** hard gap → `blocked`; WIP/gaps → `researching`; below bar → `passed`; economics clear → match buyers → empty book `watch` | fits `ranked` | no fit `passed`.

**Expenses:** Fix-up costs (Acquisition CapEx) → `cash_in` via the **`cost-estimate`** skill; Monthly costs (Recurring OpEx) → `monthly_opex_total` via the **`opex`** skill. Never bump `maintenance_pct` as a CapEx/OpEx proxy.

**UI copy:** When editing the web app, write for realtors — plain words, no schema field names in labels/helpers/tips.

**Pipeline:** ingest (detail page, not search card) → research (+ trends if stale) → triage → scenarios → underwrite.

**Deal load:** `get_deal` defaults to **`view=core`**. Use `list_*` or `view=overview` / `view=full` for nests.

**Quick Analyze / Auto import:** enqueue via web UI or MCP `create_automation_run` → poll `get_automation_run` (do not block one call). Goal=`import` — research only (no status/`advance_deal`). Existing deal: the same tool with `deal_id` + `goal=research|underwrite` (or `kind` for a single tab refresh). Planner + persona map (Critic/Brief target): `read_doc("agents/automation-planner.md")`.

## Workflow index

| When | Skill |
|------|-------|
| Allowlisted listing connectors (catalog + portal gotchas) | `ingest` → [references/data-sources.md](../ingest/references/data-sources.md) |
| Sweeps, `add_manual_lead`, **detail page** (write-up, **full gallery**, lat/lng, facts) | `ingest` |
| Tax/HOA/rent/ADR/ARV, photos gap-fill, handoffs, **comps** | `research` |
| Qualitative demand signals (mall, employer, infra) → **Drivers** | `growth-drivers` |
| Queue batch, Blocking/Soft gaps, shortlist, cluster pass | `triage` |
| Scenario runs (select/apply decisions); dry-run Preview only via `cashflow_scenarios` | `scenarios` |
| Verdict, status, UW comment (+ hold projections); prefer **`advance_deal(dry_run=true)`** | `underwrite` |
| Market snapshots / catalog refresh / projections — **`ensure_geo_markets`** then **`refresh_deal_trends`** (scan / `gap_fill`) | `trends` |
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
| Photos | Full gallery from the **detail page** on ingest; gap-fill via the ingest / research playbooks |
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

On failure many tools return `{"error": "..."}` — read the message before retrying.

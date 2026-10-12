---
name: cost-estimate
description: >-
  Tighten RealtyPad deal CapEx (Fix-up costs) with ZIP-local Home Depot
  materials (Apify) plus public labor/project cost ranges from web search,
  then persist structured repair items. Use when setting or revising CapEx for
  fix_flip / wholesale / needs_rehab deals, or when the user says repairs,
  repair estimate, rehab budget, or fix-up costs.
---

# RealtyPad — cost / repair estimate

Evidence-only CapEx. **Never invent line items.** Materials via allowlisted Home Depot Actors; labor via the user’s web search / browse (Angi True Cost Guide, Homewyse, Fixr, local quotes). Do not set `ranked` / `passed`.

## Expenses mental model (write once)

| Bucket | Persist how | Feeds | Never | UI label |
|--------|-------------|-------|-------|----------|
| **Acquisition CapEx** | `add_deal_repair_item(s)` (scalar `repair_estimate` only if no items) | `repair_estimate` → `cash_in`; sets `needs_rehab` | Invent into OpEx / bump `maintenance_pct` | **Fix-up costs** |
| **Recurring OpEx** | `add_deal_opex_item(s)` — see the **`opex`** skill | `monthly_opex_total` → monthly CF | Day-0 rehab BOM | **Monthly costs** |

Soft gap: `needs_rehab` + null estimate → `repairs_unknown` — flag it; do not invent OpEx to clear it. When items exist, they own `repair_estimate` (scalar patches ignored).

**UI note:** Expenses tab tips/labels use plain realtor language (Fix-up / Monthly costs). Keep schema jargon in tools and manuals only. For Monthly costs checklist and STR/LTR line sets, load the **opex** playbook — this file owns CapEx only.

## Hard rules

- Persist **structured** CapEx via `add_deal_repair_item(s)` — not markdown BOM tables as source of truth.
- Writes auto-sum item `line_total`s onto `deals.repair_estimate` and set `needs_rehab=true` when items exist.
- Recurring reserves for future CapEx belong on **OpEx** (`add_deal_opex_item` category `capex_reserve`) — distinct from acquisition `repair_estimate`. Full OpEx: the **`opex`** skill.
- **Never** use `maintenance_pct` / `insurance_annual` as a blended CapEx or OpEx proxy.
- Optional short **comment** for confidence/narrative (not `add_observation(kind=estimate)`).
- HD prices = **retail materials**, not contractor invoice. Labor usually dominates paint/fence/deck packages.
- Prefer **low confidence** until a walkthrough or contractor bid exists.
- Cap connector spend with `maxTotalChargeUsd`. Do **not** call non-allowlisted Actors.
- RealtyPad automation CapEx phase may use Firecrawl for labor; chat agents without Firecrawl should leave labor as a gap and materials-only floor.
- **Hosted tab research** (`kind=capex`): the worker attaches listing gallery photos as Gemini vision inputs (up to ~12) so condition scope comes from images, not URL strings alone.

## Checklist

```
Cost estimate:
- [ ] 1. `get_deal` core + `list_deal_repair_items` — condition notes, photos via `list_deal_attachments`, existing CapEx
- [ ] 2. Scope line items from evidence (paint, fence, deck, …) — skip inventing systems
- [ ] 3. Materials: Home Depot near deal ZIP — see [home-depot.md](../ingest/references/data-sources/home-depot.md)
- [ ] 4. Labor/project ranges: web search Angi/Homewyse/Fixr (national OK; note geo)
- [ ] 5. Persist via add_deal_repair_items (replace_all if rewriting) with nested lines
- [ ] 6. Optional `add_comment` for confidence/narrative; `list_deal_repair_items` to confirm rollup
```

## Structured CapEx (MCP)

| Tool | Use |
|------|-----|
| `list_deal_repair_items` | Read current BOM + rollup |
| `add_deal_repair_item` | One scope row (+ optional `lines`) |
| `add_deal_repair_items` | Bulk (max 100); `replace_all=true` for full rewrite |
| `update_deal_repair_item` / `delete_deal_repair_item` | Edit / remove |
| `add_deal_repair_lines` | Append materials after Home Depot pull |

**Item fields:** `name` (required), `category` (paint/fence/deck/…), `hours`, totals, `notes`, `source`/`source_url`.

**Line fields:** `kind` = `material` \| `labor` \| `other`; `description`; `quantity` / `unit` / `unit_cost`; `hours`; `line_total` (or qty×unit_cost); `source_url` (HD product URL).

When lines exist, item materials/labor/`line_total`/`hours` are recomputed from children. Summary-only items (no lines) may set totals directly.

## Home Depot materials (allowlisted)

Retail materials near the deal ZIP — full recipe: **[home-depot.md](../ingest/references/data-sources/home-depot.md)** (or catalog the data-source catalog ([data-sources.md](../ingest/references/data-sources.md))). Cap with `maxTotalChargeUsd`. Map products into `add_deal_repair_lines` / item `lines` with `kind=material` + `source_url`.

## Labor / project ranges (no listing connector)

Client web search for Angi True Cost Guide (or Homewyse/Fixr). National/metro OK; label geo. Add as `kind=labor` lines (qty=hours, unit=`hour`, unit_cost=rate) or summary labor totals. Directory-only labor marketplaces are not price sources.

## Persist example shape

```
add_deal_repair_items(deal_id, replace_all=true, items=[
  {
    name: "Exterior paint",
    category: "paint",
    lines: [
      {kind: "material", description: "Behr exterior 1gal", quantity: 8, unit: "gallon", unit_cost: 71.98, source_url: "…"},
      {kind: "labor", description: "Painter crew", quantity: 24, unit: "hour", unit_cost: 55, hours: 24}
    ]
  },
  …
])
```

Flip/wholesale still need `arv` before score is meaningful. Brief: materials floor vs all-in mid, what remains unverified.

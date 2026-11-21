---
name: cost-estimate
description: >-
  Tighten RealtyPad deal CapEx (Fix-up costs) with ZIP-local Home Depot
  materials (utility_material_search) plus public labor/project cost ranges from
  web search, then persist structured repair items. Use when setting or
  revising CapEx for fix_flip / wholesale / needs_rehab deals, or when the user
  says repairs, repair estimate, rehab budget, or fix-up costs.
---

# RealtyPad — cost / repair estimate

Evidence-only CapEx. **Never invent line items.** Materials come from `utility_material_search` (Home Depot retail, read-only); labor from the user’s web search / browse (Angi True Cost Guide, Homewyse, Fixr, local quotes). Do not set `ranked` / `passed`.

## Expenses mental model (write once)

| Bucket | Persist how | Feeds | Never | UI label |
|--------|-------------|-------|-------|----------|
| **Acquisition CapEx** | `deal_item_upsert(kind=repair)` (scalar `repair_estimate` only if no items) | `repair_estimate` → `cash_in`; sets `needs_rehab` | Invent into OpEx / bump `maintenance_pct` | **Fix-up costs** |
| **Recurring OpEx** | `deal_item_upsert(kind=opex)` — see the **`opex`** skill | `monthly_opex_total` → monthly CF | Day-0 rehab BOM | **Monthly costs** |

Soft gap: `needs_rehab` + null estimate → `repairs_unknown` — flag it; do not invent OpEx to clear it. When items exist, they own `repair_estimate` (scalar patches ignored).

**UI note:** Expenses tab tips/labels use plain realtor language (Fix-up / Monthly costs). Keep schema jargon in tools and manuals only. For Monthly costs checklist and STR/LTR line sets, load the **opex** playbook — this file owns CapEx only.

## Hard rules

- Persist **structured** CapEx with `deal_item_upsert(kind=repair)` rows — not markdown BOM tables as source of truth.
- Writes auto-sum item `line_total`s onto `deals.repair_estimate` and set `needs_rehab=true` when items exist.
- Recurring reserves for future CapEx belong on **OpEx** (`kind=opex`, category `capex_reserve`) — distinct from acquisition `repair_estimate`. Full OpEx: the **`opex`** skill.
- **Never** use `maintenance_pct` / `insurance_annual` as a blended CapEx or OpEx proxy.
- Optional short **comment** (`comment_create`) for confidence/narrative (not an `observation_upsert` note).
- Home Depot prices = **retail materials**, not contractor invoice. Labor usually dominates paint/fence/deck packages.
- Prefer **low confidence** until a walkthrough or contractor bid exists.
- Material search is one provider run, cached a day; `refresh=true` re-runs it, so use it only with a reason. Keep `max_results` small.
- Chat agents without web search should leave labor as a gap and give a materials-only floor.
- **Condition scope from photos.** `utility_image_analyze(purpose=capex, deal_id)` (read-only, at most 12 photos) returns room-by-room repair signals, each citing its photo. Use it as evidence for the scope, not as a price. Hosted tab research (`kind=capex`) does the same.

## Checklist

```
Cost estimate:
- [ ] 1. `deal_get` core + `deal_item_list(kind=repair)` — condition notes, photos via `deal_item_list(kind=attachment)`, existing CapEx
- [ ] 2. Scope line items from evidence (listing write-up, `utility_image_analyze(capex)`: paint, fence, deck, …) — skip inventing systems
- [ ] 3. Materials: `utility_material_search(query, zip_code)` per scope item
- [ ] 4. Labor/project ranges: web search Angi/Homewyse/Fixr (national OK; note geo)
- [ ] 5. Persist with `deal_item_upsert(kind=repair)` rows and nested `lines` (replace = delete then upsert)
- [ ] 6. Optional `comment_create` for confidence/narrative; `deal_item_list(kind=repair)` to confirm the roll-up
```

## Structured CapEx (MCP)

| Tool | Use |
|------|-----|
| `deal_item_list(deal_id, kind=repair)` | Read current BOM + roll-up (`summary.repair_estimate`) |
| `deal_item_upsert(deal_id, items=[{kind: "repair", …}])` | Create scope rows (max 100 per call, with optional `lines`); a row with `id` patches only the fields you send |
| `deal_item_delete(deal_id, kind=repair, ids)` | Remove rows. To rewrite the whole BOM: list, delete the ids, then upsert the new rows |
| `utility_material_search(query, zip_code, store_id?, max_results?)` | Read-only Home Depot products near the ZIP |

**Item fields:** `name`, `category` (paint/fence/deck/…), `hours`, totals, `notes`, `source`/`source_url`.

**Line fields** (`lines` on an item): `kind` = `material` \| `labor` \| `other`; `description`; `quantity` / `unit` / `unit_cost`; `hours`; `line_total` (or qty×unit_cost); `source_url` (product URL).

When lines exist, item materials/labor/`line_total`/`hours` are recomputed from children. Summary-only items (no lines) may set totals directly. **Adding materials to an existing item replaces its nested lines:** read the item with `deal_item_list`, then send the full set of lines (old and new) in the update.

## Materials (`utility_material_search`)

Retail materials near the deal ZIP. Give `query` (the product, for example "exterior paint 1 gallon") and `zip_code`. The search localizes by Home Depot store, not ZIP: pass `store_id` for a store near the ZIP, or the result warns that the default store was used. The result is a list of products (`name`, `brand`, `model`, `price` in dollars, product URL, stock, rating), each with a `source_id`. Material results are not saved by id: copy the chosen product into a `kind=material` line (`description`, `quantity`, `unit`, `unit_cost`, `source_url`). Do not invent a price a search did not return.

## Labor / project ranges (no search tool)

Client web search for Angi True Cost Guide (or Homewyse/Fixr). National/metro OK; label geo. Add as `kind=labor` lines (qty=hours, unit=`hour`, unit_cost=rate) or summary labor totals. Directory-only labor marketplaces are not price sources.

## Persist example shape

```
deal_item_upsert(deal_id, items=[
  {
    kind: "repair",
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

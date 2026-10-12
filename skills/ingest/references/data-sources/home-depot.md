# Home Depot (materials connector)

## Role

Retail **materials** prices for Acquisition CapEx / repair BOM near the deal ZIP. Labor is **not** from these Actors — use Angi / Homewyse / Fixr via client web search.

## Lead source

N/A (materials lines, not listing capture).

## Tools

| Job | Actor id | MCP tool |
|-----|----------|----------|
| Keyword → store prices | `sian.agency/home-depot-product-scraper` | `sian-agency--home-depot-product-scraper` |
| SKU + ZIP confirm | `maplerope44/home-depot-product-lookup` | `maplerope44--home-depot-product-lookup` |

## Input recipe

1. `fetch-actor-details` → confirm schema.
2. **Sian first** — tight single keywords, small `maxItems`, `storeId` when known (78745 trial: `6570`).
3. Optional **maplerope** ZIP confirm for known product IDs.
4. Always set `maxTotalChargeUsd`.

## Gotchas

- HD prices = **retail materials**, not contractor invoice. Labor usually dominates paint/fence/deck packages.
- Do **not** call non-allowlisted Actors.
- Thumbtack / directory sites are not price sources for labor.
- Prefer low confidence until a walkthrough or contractor bid exists.

## Persist hints

Map products into `add_deal_repair_lines` / item `lines` with `kind=material` + `source_url` (product URL). Full CapEx playbook: the **`cost-estimate`** skill.

## Related

- Cost / repair estimate: the **`cost-estimate`** skill
- Catalog: the data-source catalog ([data-sources.md](../data-sources.md))

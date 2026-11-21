---
name: growth-drivers
description: >-
  Log qualitative demand signals (new employers, malls, transit,
  infrastructure, rezoning, closures) as RealtyPad deal growth drivers with
  sources, strength and direction, scoped to the right market. Use when the
  user asks for growth drivers, demand signals, what's happening in the area,
  local news/planning research, or when the missing_growth_drivers soft gap
  must clear before research or import is complete.
---

# RealtyPad — growth drivers (qualitative demand signals)

## Hard rules

- **Never invent.** Every driver needs a real source (`source` + preferably `source_url`). Prefer null / skip over fiction. A driver from `utility_growth_driver_search` already cites a URL the search returned.
- **Do not** log “prices are rising” or YoY chips — that is the **`trends`** skill / market **Growth**. Log the **cause** (mall, employer, transit, rezoning, etc.).
- Prefer **`status=dismissed`** over hard delete when a story goes stale or was wrong.
- Read existing drivers via `deal_item_list(kind=growth_driver)` (or `deal_get` overview/full) before re-searching; update/dismiss instead of duplicating.
- **Search is read-only; saving is separate.** `utility_growth_driver_search` saves nothing. Save the drivers you keep with `deal_item_upsert(deal_id, kind=growth_driver, source_ids)`.
- Do **not** set triage status / Status note — that is underwrite/triage.
- After leaving inbox (`status != new`), deals with **no active** drivers emit soft gap `missing_growth_drivers` (`has_gap=missing_growth_drivers` / Soft gaps queue). Dismissed-only does not clear it — add or reactivate an active driver.
- **Scope each driver to the area it affects.** A project can matter to one neighborhood, a ZIP, the city, the county or the whole metro. The search tags each driver with its area (`area` = `level:code`, `area_name`) and the save sets `market_id` to the deal's linked market for that area (none when the area is not linked). For drivers you write by hand, set `market_id` to the deal's linked market for the **smallest** area it actually affects (`deal_get` → `markets[]`); omit only when unclear. A street-level project is the neighborhood / ZIP; a citywide policy is the city; a regional employer, highway or transit line is the county or metro.
- **Automation fact step:** hosted `import` / `research` runs search and grade drivers in code before the planner starts (every linked area searched in parallel, each driver scoped to its area, every driver cites a search-result URL, cached 14 days). The worker only researches drivers when that step found none or the ones logged don't fit this property — dismiss off-target ones rather than deleting.
- **Ready package (import / research automation):** Hosted `goal=import` and `goal=research` must leave at least one **active** growth driver before `done`. Soft gap stays soft for UW list filters, but when the planner calls `done` with the gap open, a code rail sends the worker to research a growth driver first (once), then the critic done gate decides. Prefer real sourced drivers after trends are fresh — never invent.

## Owns vs does not

| Owns | Does not own |
|------|----------------|
| Searching + logging qualitative demand signals | Market YoY / `trend_refresh` (`trends`) |
| Grading strength × direction; dismissing stale items | Money fields / comps / tax (`research`) |
| Feeding **Drivers** into Overall via CRUD auto-rescore | Status / aggressiveness / buyer gate (`underwrite`) |

## What a growth driver is

Deal-scoped evidence that could move future demand, rents, or values **beyond** what ZHVI/ZORI/AirROI already show:

- New retail / residential / mixed-use development
- Named employer relocating or expanding nearby
- Infrastructure (highway, transit, airport, utilities)
- Policy / zoning / school bonds / tax incentives
- Amenities (park, hospital, university)
- Demographic shifts (migration, age mix) with a cited report
- Other sourced signals

## Search playbook

Run after trends are fresh (`market_ensure` + `trend_refresh` gap-fill as needed).

**Start with `utility_growth_driver_search(deal_id)`** (or `address` with city and state). It searches every area linked to the deal (neighborhood, ZIP, city, county, metro), grades each driver, and returns items with `source_id`, `title`, `category`, `strength`, `direction`, `summary`, `source`, `source_url`, `area`, `as_of`. Use `levels` to keep some areas. Results are cached about 14 days (an empty result an hour): `refresh=true` only when the cache is clearly wrong. Review the items against the deal, then save the ones that fit. Then search by hand only for gaps, using deal address / city / county:

1. **Local news** — “{city} development”, “{employer} relocating {county}”, mall / industrial park announcements.
2. **City / county planning** — permits, rezoning agendas, CIP / bond packages.
3. **Chamber / economic development** — employer wins, site selectors.
4. **Transit / infrastructure** — DOT, transit authority, airport expansions.
5. **Schools / hospitals / universities** — campus expansions when they move housing demand.

Prefer Firecrawl or the user’s own MCP sources. Respect robots.txt; no ToS-hostile scrapes without approval.

## Grading

Two axes on every row:

| Axis | Values | Meaning |
|------|--------|---------|
| **strength** | `strong` / `weak` / `neutral` | How credible and material |
| **direction** | `positive` / `negative` | Expected effect on demand/price for **this** asset class |

**Strength guide**

| Strength | Use when |
|----------|----------|
| `strong` | Funded / under construction / permitted; large named employer with timeline; bond passed |
| `weak` | Early rumor, RFP, soft LOI, “exploring”, unfunded concept |
| `neutral` | Noted for context; impact unclear or balanced |

**Direction:** positive = supports rents/values/demand for this strategy; negative = headwind (plant closure, flood plain expansion, competing oversupply if sourced).

**Points (auto):** strong ±100, weak ±40, neutral 0. **Drivers** = mean of **active** rows (−100…+100). Dismissed rows excluded.

## Scoring (do not hand-edit)

Overall (when both Growth and Drivers present):

`econ_adj + 0.35×Growth + 0.15×Drivers`

Growth only → `0.5×Growth` (legacy). Drivers only → `0.25×Drivers`. Growth chip stays market YoY; Drivers is separate.

## Checklist

```
Growth drivers:
- [ ] 1. `deal_get` core for `score_summary.drivers` + `deal_item_list(kind=growth_driver)`
- [ ] 2. Trends already fresh (or call trends workflow first)
- [ ] 3. `utility_growth_driver_search(deal_id)`; then news / planning / chamber / infra by hand for gaps (cite URLs)
- [ ] 4. Save: `deal_item_upsert(kind=growth_driver, source_ids)` for search items; hand-found drivers as `items` rows with title, category, strength, direction, source(+url), as_of, market_id (area it affects)
- [ ] 5. Dismiss stale/duplicate drivers (`status=dismissed` on the row's `id`)
- [ ] 6. Brief: what logged, Drivers score, what still speculative
```

## MCP

| Step | Tool |
|------|------|
| Load | `deal_get` (core), `deal_item_list(kind=growth_driver)` (dismissed rows included by default; `include_dismissed=false` hides them) |
| Search | `utility_growth_driver_search(deal_id \| address, levels?)` (read-only) |
| Add | `deal_item_upsert(deal_id, kind=growth_driver, source_ids)` for search items, or `items=[{kind: "growth_driver", …}]` for hand-found ones (max 100). All ids are written or none: on `source_expired`, search again |
| Update / dismiss | `deal_item_upsert(items=[{kind: "growth_driver", id, status: "dismissed"}])` |
| Hard delete | `deal_item_delete(kind=growth_driver, ids)` (rare) |

**Categories:** `development` | `employer` | `infrastructure` | `policy` | `amenity` | `demographic` | `other`

**UI:** Deal detail **Growth drivers** tab. Also on Home summary.

## Anti-patterns

- Logging “ZHVI up 5%” as a driver
- Inventing an employer move without a URL
- Marking rumors as `strong`
- Leaving obsolete “coming soon” malls as `active` for years — dismiss them

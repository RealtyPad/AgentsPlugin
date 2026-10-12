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

- **Never invent.** Every driver needs a real source (`source` + preferably `source_url`). Prefer null / skip over fiction.
- **Do not** log “prices are rising” or YoY chips — that is the **`trends`** skill / market **Growth**. Log the **cause** (mall, employer, transit, rezoning, etc.).
- Prefer **`status=dismissed`** over hard delete when a story goes stale or was wrong.
- Read existing drivers via `list_deal_growth_factors` (or `get_deal` overview/full) before re-searching; update/dismiss instead of duplicating.
- Do **not** set triage status / Status note — that is underwrite/triage.
- After leaving inbox (`status != new`), deals with **no active** drivers emit soft gap `missing_growth_drivers` (`has_gap=missing_growth_drivers` / Soft gaps queue). Dismissed-only does not clear it — add or reactivate an active driver.
- **Scope each driver to the area it affects.** A project can matter to one neighborhood, a ZIP, the city, the county or the whole metro. Set `market_id` to the deal's linked market for the **smallest** area it actually affects (`get_deal` → `markets[]`); omit only when unclear. A street-level project is the neighborhood / ZIP; a citywide policy is the city; a regional employer, highway or transit line is the county or metro.
- **Automation fact step:** hosted `import` / `research` runs search and grade drivers in code before the planner starts (every linked area searched in parallel, each driver scoped to its area, every driver cites a search-result URL, cached 14 days). The worker only researches drivers when that step found none or the ones logged don't fit this property — dismiss off-target ones rather than deleting.
- **Ready package (import / research automation):** Hosted `goal=import` and `goal=research` must leave at least one **active** growth driver before `done`. Soft gap stays soft for UW list filters, but when the planner calls `done` with the gap open, a code rail sends the worker to research a growth driver first (once), then the critic done gate decides. Prefer real sourced drivers after trends are fresh — never invent.

## Owns vs does not

| Owns | Does not own |
|------|----------------|
| Searching + logging qualitative demand signals | Market YoY / `refresh_deal_trends` (`trends`) |
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

Run after trends are fresh (`ensure_geo_markets` + gap-fill as needed). Use deal address / city / county:

1. **Local news** — “{city} development”, “{employer} relocating {county}”, mall / industrial park announcements.
2. **City / county planning** — permits, rezoning agendas, CIP / bond packages.
3. **Chamber / economic development** — employer wins, site selectors.
4. **Transit / infrastructure** — DOT, transit authority, airport expansions.
5. **Schools / hospitals / universities** — campus expansions when they move housing demand.

Prefer Firecrawl or allowlisted listing/MCP sources. Respect robots.txt; no ToS-hostile scrapes without approval.

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
- [ ] 1. `get_deal` core for `score_summary.drivers` + `list_deal_growth_factors`
- [ ] 2. Trends already fresh (or call trends workflow first)
- [ ] 3. Search news / planning / chamber / infra (cite URLs)
- [ ] 4. add_deal_growth_factor(s) with title, category, strength, direction, source(+url), as_of, market_id (area it affects)
- [ ] 5. Dismiss stale/duplicate drivers (status=dismissed)
- [ ] 6. Brief: what logged, Drivers score, what still speculative
```

## MCP

| Step | Tool |
|------|------|
| Load | `get_deal` (core), `list_deal_growth_factors` |
| Add | `add_deal_growth_factor` / `add_deal_growth_factors` (bulk ≤100) |
| Update / dismiss | `update_deal_growth_factor` (`status=dismissed`) |
| Hard delete | `delete_deal_growth_factor` (rare) |

**Categories:** `development` | `employer` | `infrastructure` | `policy` | `amenity` | `demographic` | `other`

**UI:** Deal detail **Growth drivers** tab. Also on Home summary.

## Anti-patterns

- Logging “ZHVI up 5%” as a driver
- Inventing an employer move without a URL
- Marking rumors as `strong`
- Leaving obsolete “coming soon” malls as `active` for years — dismiss them

---
name: ingest
description: >-
  Pull market listings into RealtyPad from any available connector (Apify
  listing connectors, Firecrawl, user scraping MCP, or MLS MCP), normalize,
  upsert via add_manual_leads (batches) or add_manual_lead (single), attach
  photos, capture listing_description + lat/lng. Includes the allowlisted
  data-source catalog and per-portal recipes (Zillow, HUD, Auction.com,
  Redfin, Realtor.com, Trulia, Craigslist, probate/foreclosure, VRBO, AirDNA,
  Home Depot). Use when the user asks for a sweep, scrape, ingest,
  HUD/Zillow/Auction/Craigslist pull, MLS import, market lead pull, listing
  enrich/backfill, or which data sources are available.
---

# RealtyPad — ingest (source-agnostic)

## Hard rules

- **Never invent numbers.** Null > fiction. Mark estimates.
- Dedupe on normalized **street + unit + ZIP5** (`properties.address_key`, tenant-scoped). Always pass `postal_code` so re-runs update the same deal. Listing URLs (`source_url` / `listing_urls`) still attach and stay unique per workspace; URL match is a **fallback** when ZIP is missing. Capture source (`lead_source_code`) stays frozen on merge.
- **Trust but verify on re-ingest / dedupe.** Pass the portal’s **current** ask (and listing status when shown). Do not leave a stale stored `ask_price` after a price cut or re-list — compare portal price → stored price → write the new value when different and note the change.
- Re-ingest of an **archived** deal updates money fields but **does not unarchive** (`archived=true` on bulk items). Explicit `update_deal(archived=false)` to restore.
- Do **not** pull from ToS-hostile sites without explicit user approval (the **`realtypad`** skill).
- Cap spend on paid connectors before running.
- Prefer **public/gov** sources first; MLS when the user has a connected MLS MCP / credentials.
- **Stop after ingest + detail enrich** when the human only asked for ingest (manual MCP / sheet). **Hosted `goal=import`** (Quick analyze / `create_automation_run`) continues into the research pack pool and must produce a **go/no-go ready package** — including qualitative growth drivers — before automation `done` (see the **`growth-drivers`** skill / `automation-planner`). Research, triage, and underwrite remain separate when run as their own goals.
- **Search cards are not ingest.** A Zillow/HUD/auction search hit (price, beds, one `imgSrc`) is a pointer. Open the **detail page** (or run the detail Actor) and persist everything useful that is already there before calling the sweep done.

## Connector selection

Before picking a paid connector, **search/discover** what portals and public pages already cover the geography (Firecrawl/web search, MLS if connected, gov lists). Then pick tools that fill gaps — prefer multiple sources for completeness over a single Actor. Full catalog + gotchas: the data-source catalog ([data-sources.md](references/data-sources.md)).

| Connector | When to use | How |
|-----------|-------------|-----|
| **User MLS MCP** | User has MLS connected; board-compliant pull | That MCP’s search/listing tools → normalize |
| **User listing MCP** | User points at their own Actor/tool | Call their tools; same normalize path |
| **Listing connectors (platform)** | Default public portal sweeps (HUD, Auction, Zillow, CL, …) | RealtyPad MCP dedicated tools; see `data-sources` |
| **Hosted Quick analyze** | Address or listing URL first pass (web or agent) | MCP `create_automation_run` → poll `get_automation_run`; goal=`import` planner (listing → research phases including growth drivers). URL host tries matching portal first. Property/photos refresh runs only when ingest left gaps — use deal-tab research or `create_automation_run(goal=research, deal_id=…)` for re-pulls. |
| **Firecrawl** | Single URL / small batch enrich, or search pages | Firecrawl MCP / scrape tools |
| **Manual / sheet** | User pastes URLs or a small list | Normalize → `add_manual_leads` (or single `add_manual_lead`) |

If multiple connectors exist, prefer **MLS MCP** for MLS inventory and **platform connectors / Firecrawl** for public distress / FSBO / HUD. Ask once if spend or ToS is ambiguous.

## Workflow

```
Ingest Progress:
- [ ] 1. Confirm geography, income_strategy, price ceiling, lead_source, connector
- [ ] 2. Pull search/list results (see connector sections)
- [ ] 3. Detail pull each keep — Zillow/HUD/auction/MLS listing page or detail Actor (not the search card)
- [ ] 4. Normalize → add_manual_leads (batch; shared defaults + image_urls + latitude/longitude + listing_description + amenities + source_url + listing_urls for every portal found)
- [ ] 5. Gap-fill only: add_deal_attachment_urls if gallery was thin / missing on create
- [ ] 6. Confirm enrich: description + lat/lng + full gallery + beds/baths/sqft/year/lot + amenities + deal_listings (primary + extras)
- [ ] 7. Brief counts + standouts (do not auto-research/UW unless asked)
```

## Normalize → upsert

Prefer MCP `add_manual_leads` for Apify/batch sweeps (max 100; compact results). Use `add_manual_lead` for one-offs. Shared defaults (`income_strategy`, `lead_source_code`, `status`, …) fill unset item fields. Per item: optional `latitude`/`longitude` (skip Nominatim), **`image_urls`** (full gallery, first = primary, max 40), **`listing_description`**, **`amenities`**, and **`listing_urls`**. `primary_image_url` is hero-only fallback when `image_urls` is omitted. On address (or URL-fallback) match returns `deduped: true`. A match merges onto the existing deal by default (price drops on re-sweeps update the ask); REST `POST /api/deals` also takes `on_duplicate=skip`, which returns the existing deal with `deduped: true` and changes nothing on it (the human Add deal path). Response includes `photos_attached` / `photo_errors` (soft-fail — deal still created).

**Dedupe reingest is not a photo skip path.** Address/URL matches still run the same gallery attach as create (`_finish_manual_lead`). If `photos_attached` is 0, you omitted `image_urls` (or attach soft-failed) — fix in the same pass. Leaving `status=new` without photos does not emit a blocking `photos` data_gap yet; once the deal leaves the inbox, `data_gaps` includes `{ code: "photos", blocking: true }` (filter: `has_gap=photos`).

| Field | Source |
|-------|--------|
| `address_line1`, `address_line2` (unit), `city`, `state`, `postal_code` | Listing — **`postal_code` required for address dedupe** |
| `ask_price` | List / **starting bid** (say which in notes) |
| `estimated_rent` | Only if present; else null |
| `beds`, `baths`, `sqft`, `year_built`, `lot_size_acres` | Listing |
| `source_url` | **Primary** listing URL — attached on merge; URL fallback when ZIP missing |
| `listing_urls` | Extra marketplace URLs (`[{url, lead_source_code?}]`) when also on Zillow/Auction/etc. |
| `image_urls` | Full listing photo set from connector (prefer on create) |
| `listing_description` | Verbatim listing remarks (prefer on create) |
| `amenities` | Listing amenity codes (`pool`, `garage`, `ac`, `fenced_yard`, …) when shown |
| `lead_source_code` | **Capture** source: `hud`, `zillow`, `auction_com`, `craigslist`, `redfin`, `realtor_com`, `trulia`, `vrbo`, `har`, `mls`, `realtor`, … (`list_lead_sources`) |
| `income_strategy` | `ltr` (default), `str`, `fix_flip`, or `wholesale` |
| `auction_date` | Auction / sale day when known (distress) |
| `liens_owed` | Delinquent tax / liens when known — never invent |
| `notes` | Connector run id / dataset id / MLS query, capture caveats |

Default status for new manual leads is **`new`** (inbox). Leave it unless the user wants an immediate watch book (`status=watch`) or you are continuing active research (`researching`). Do **not** default ingest to `researching`.

Do **not** stuff lead source into titles. Do **not** invent rent/tax to force a score.

## Photos

Pass the **full gallery on create** via `image_urls` (not just the hero):

1. Collect image URLs from the **detail** payload (portal photo arrays). Cap ~40 per deal if huge. Do **not** expect search Actors with `fetchDetails: true` to fill galleries — that field is often empty (see [Zillow](references/data-sources/zillow.md)).
2. Include them on `add_manual_leads` / `add_manual_lead` as `image_urls` (first = primary). Skip map/satellite Google URLs. **Re-sweeps / dedupe must pass `image_urls` again** when the deal still lacks a gallery — price-only reingest leaves `photos_attached=0`.
3. Use `add_deal_attachment_urls` only to **gap-fill** when create had no/thin photos. Default Zillow gap-fill: detail Actor with **`addresses`** — [Zillow](references/data-sources/zillow.md). Exact `source_url` on the deal is deduped.
4. Confirm via `list_deal_attachments` / `photos_attached` that count matches the listing photo count (within cap). Treat `photos_attached=0` on the create/dedupe response as a hard incompleteness signal — not “photos optional.”

## Detail pull is required (not optional light enrich)

Search connectors (Zillow PPE, HUD lists, auction search) usually return a **card**: URL, price, beds, maybe one photo. That is not enough to underwrite later. For every lead you keep, run a **detail** fetch and map every listed fact you can persist. Portal recipes: the data-source catalog ([data-sources.md](references/data-sources.md)).

| Job | How |
|-----|-----|
| Zillow (default) | Detail Actor with **`addresses`** — [Zillow](references/data-sources/zillow.md). Firecrawl the homedetails URL is an alternative. Cap `maxTotalChargeUsd`. |
| HUD / auction / other portals | Firecrawl or that portal’s detail Actor — [HUD](references/data-sources/hud.md), [Auction.com](references/data-sources/auction-com.md), [Trulia](references/data-sources/trulia.md), … |
| MLS MCP | Listing detail + media from the feed; skip a second pull when the feed already has remarks + photos + lat/lng. |

### What to persist from the detail payload

Copy values that are **on the listing**. Never invent. Never rewrite marketing copy in your own voice.

| Persist | Tool | Typical Zillow detail keys (same idea on other portals) |
|---------|------|----------------------------------------------------------|
| Listing write-up | `listing_description` on create (or `update_deal` / `update_deals` to gap-fill) | `description`, `homeDescription`, `remarks` (verbatim) |
| Amenities | `amenities` on create (or `update_deal_property`) | listing feature flags — closed catalog codes only — never invent |
| Property attributes | `attributes` on create or `update_deal_property` / hosted automation (JSONB bag) | Interior/Exterior/parking/subdivision/`mls_id`/… from the detail grid — soft convention keys (glossary). Agents research + patch; **do not** build portal feature adapters. HOA → `hoa_monthly` money field |
| Map pin | `add_manual_leads` `latitude`/`longitude`; else `update_deal_property` `lat`/`lng` | `location.latitude` / `location.longitude`, `latitude`, `longitude` |
| Full gallery | `image_urls` on create (first = primary); gap-fill `add_deal_attachment_urls` | `media.allPropertyPhotos.highResolution`, `photos[]` / `responsivePhotos[]` — skip Google map/satellite |
| Ask / bid | `ask_price` | `price`, `listPrice`; auction = starting/live bid (say which in notes) |
| Beds / baths / sqft / year / lot | create fields or `update_deal_property` | `bedrooms`, `bathrooms`, `livingArea`, `yearBuilt`, `lotSize` (lot sqft → acres ÷ 43560) |
| HOA if **listed** | `update_deal` `hoa_monthly` | `resoFacts.hoaFee`, `monthlyHoa` — only if present |
| Tax if **listed** | `update_deal` `tax_annual` + `add_deal_appraisal_tax_history` when the payload has a year array | listing tax **and** `_details.taxHistory` / `listingTaxHistory` (assessor-sourced — persist now; research still checks homestead jumps vs listing quote) |
| Listing agent | **`link_deal_contact`** `role=listing_agent` (phone/email/broker when listed); optional `realtor_name` on create as a raw mirror only | `attributionInfo.agentName` / `agentPhoneNumber` / `brokerName`, `listedBy` |
| Extra portals | `listing_urls` | same house on Auction.com / Realtor.com / etc. |

**Listing agent is a contact, not a string.** When the detail payload names an agent, create+link via `link_deal_contact` (`role=listing_agent`) in the same ingest pass — pass **phone and/or email** whenever the portal shows them so the directory reuses the same person across deals (`deduped=true`). Prefer `list_contacts(q=phone_or_email)` then `link_deal_contact(contact_id=…)` when you already know the agent. Do not stop at `realtor_name` / an observation. Skip only when the listing has no agent attribution (FSBO, auction-only, etc.) and note that in the ingest brief. Cleanup: `merge_contacts` / `delete_contact` (or UI contact detail).

Do **not** treat Zestimate as rent or ARV. If you store it, put it in a **comment** or `raw` marked estimate — not an observation.

```
Enrich (ingest is incomplete until these pass or you document why not):
- [ ] listing_description non-empty (never invent marketing copy)
- [ ] amenities when the listing shows them (catalog codes — never invent)
- [ ] attributes when the detail grid shows Interior/Exterior/parking/MLS facts (JSONB bag — research + patch; no portal feature adapters)
- [ ] source_url + listing_urls for every portal detail URL found (Zillow/Redfin/Realtor/Trulia/Auction/HUD) — not hero-only / search-card links
- [ ] lat + lng present (prefer portal/MLS coords over Nominatim for new builds)
- [ ] gallery attached (full listing photo set, not hero-only)
- [ ] beds/baths/sqft/year_built/lot when the detail page shows them
- [ ] listing_agent contact linked (`link_deal_contact`) when attribution exists — else note FSBO/no agent
- [ ] raw.geocode = ok | nominatim | portal_detail | mls | failed
```

**New construction** often fails Nominatim — prefer portal/MLS coordinates. Do not invent lat/lng.

Deep tax/HOA/rent **verification** (assessor, comps, quotes) is the **`research`** skill, not this workflow. Listed tax/HOA on the detail page may still be copied now so research is not starting from a blank card.

## Listing connectors (default public sweeps)

Use **RealtyPad MCP** platform listing-connector tools (mounted without a vendor prefix).

**Allowlist only.** Catalog, run pattern, and portal gotchas: the data-source catalog ([data-sources.md](references/data-sources.md)) then ``references/data-sources/<portal>.md``. Generic `call-actor` / unrestricted web-browser Actors are **not** available. If a needed tool is missing, stop and tell the user.

| Job | Detail page |
|-----|-------------|
| HUD / REO | [hud.md](references/data-sources/hud.md) |
| Auction.com | [auction-com.md](references/data-sources/auction-com.md) |
| Open-data distress | [probate-foreclosure.md](references/data-sources/probate-foreclosure.md) |
| FSBO / Craigslist | [craigslist.md](references/data-sources/craigslist.md) |
| Zillow search → detail | [zillow.md](references/data-sources/zillow.md) |
| Redfin search | [redfin.md](references/data-sources/redfin.md) |
| Realtor.com | [realtor-com.md](references/data-sources/realtor-com.md) |
| Trulia search → detail | [trulia.md](references/data-sources/trulia.md) |
| AirDNA / VRBO (STR; not sale ingest) | [airdna.md](references/data-sources/airdna.md), [vrbo.md](references/data-sources/vrbo.md) |
| Home Depot materials | [home-depot.md](references/data-sources/home-depot.md) |

Always set `maxTotalChargeUsd` on runs. Prefer Firecrawl / listing MCP for generic page fetch — never Apify RAG browser. **Booking.com is not allowlisted.**

## MLS / user MCP connector

1. Discover tools via that namespace’s schema (search / listing / photos).
2. Pull within the user’s board rules and rate limits.
3. Map fields into the normalize table; set `lead_source_code` appropriately (`mls` or board-specific if seeded).
4. Prefer MLS description + lat/lng from the feed over a second portal pull.

## After ingest

- Brief: inserted / deduped counts, spend, 2–3 standouts.
- Confirm enrich coverage: description / coords / **full** gallery / property facts. If a keep is still card-only, it is not ingested.
- **Standouts by aggressiveness** (see the **`realtypad`** skill): LTR/STR → distress or positive-yield worth a watch; flip/wholesale → only clear asymmetry.
- **Stop.** Next steps are the **`research`** skill / the **`triage`** skill / the **`underwrite`** skill only if the user asks.

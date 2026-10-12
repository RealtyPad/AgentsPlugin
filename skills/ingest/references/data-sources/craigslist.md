# Craigslist (listing connector)

## Role

FSBO / Craigslist housing posts for a geography.

## Lead source

`craigslist`

## Tools

| Job | Actor id | MCP tool |
|-----|----------|----------|
| Craigslist | `parsebird/craigslist-scraper` | `parsebird--craigslist-scraper` |

## Input recipe

- Cap records and `maxTotalChargeUsd`.
- Confirm location / category filters via `fetch-actor-details`.

## Gotchas

- Heavy rental / spam noise — triage aggressively.
- Posts are often thin; expect manual detail / Firecrawl for address, photos, and ask clarity.
- Many FSBO posts have no listing agent — note that in the ingest brief; skip `listing_agent` contact.

## Persist hints

Normalize with `lead_source_code=craigslist`, `source_url` = post URL, gallery when available.

## Related

- Ingest: the **`ingest`** skill
- Catalog: the data-source catalog ([data-sources.md](../data-sources.md))

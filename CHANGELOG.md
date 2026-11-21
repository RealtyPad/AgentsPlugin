# Changelog

## 0.3.0

Skills move to the RealtyPad MCP v2 tool surface (ADR-0011, epic DEV-758; issue DEV-768). Use this release with the v2 connector. v1 tool names no longer appear in any skill.

### Renamed

- **Skill `ingest` is now `import`** (`realtypad:ingest` → `realtypad:import`). Use "import" everywhere; the skill directory, `name` and every cross-reference changed. Anything that loads `realtypad:ingest` by name must load `realtypad:import`.

### Removed

- The per-portal recipes and the data-source catalog (`skills/ingest/references/data-sources.md` and `data-sources/*.md`: Zillow, HUD, Auction.com, Redfin, Realtor.com, Trulia, Craigslist, probate/foreclosure, VRBO, AirDNA, Home Depot). A listing source is now the `source` parameter of `utility_listing_search` (`zillow`, `redfin`, `realtor`, `trulia`, `hud`, `auction`, `craigslist`); the `import` skill lists the filters each takes.
- Every Apify actor name and actor-run step from the skills (`fetch-actor-details`, `get-dataset-items`, `maxTotalChargeUsd`, per-actor inputs).

### Changed

- Search → pick → write by `source_id`: `utility_listing_search` / `utility_listing_get` → `deal_import`; `utility_comp_search` and `utility_growth_driver_search` → `deal_item_upsert(source_ids)`; `utility_property_get` → `deal_update(property_source_id)`. The `utility_*` tools only read.
- Deal tools: `deal_get`, `deal_list`, `deal_update` (batch rows; a status row stands alone), `deal_advance`, `deal_rescore`. Child records (comps, opex, repair, listing, listing events, appraisal/tax, attachments, growth drivers): `deal_item_list`, `deal_item_upsert`, `deal_item_delete` with `kind`. Replacing a table is delete, then upsert.
- Trends: `trend_get`, `trend_refresh` (long runs return a job: `job_get`), `trend_override_upsert`; geographies via `market_ensure` / `market_list`.
- Scenarios: `scenario_run` (also dry run and fork), `scenario_get`, `scenario_select`, `scenario_apply`, `scenario_project`.
- Buyers, contacts, notes, checks, defaults: `buyer_*`, `contact_*`, `comment_*`, `observation_upsert`, `uw_check`, `uw_default_get` / `uw_default_update`, `financing_profile_*`, `lead_source_list`, `automation_run_*`.
- `cost-estimate` uses `utility_material_search` for Home Depot materials and `utility_image_analyze` for photo condition evidence.
- Glossary: the tool tag key is `meta.fastmcp.tags` (was `meta._fastmcp.tags`) and the tag list matches v2.
- Manifest `version` is `0.3.0` in every manifest (minor bump for the rename).

### Upgrade notes

- The MCP URL in `mcp.json` / `.mcp.json` is unchanged. v2 is served at `/v2` on the API image until the dedicated v2 host is live; point the connector there to use these skills.
- RealtyPad pins this repo by tag (`AGENTS_PLUGIN_REF` in its `api/Dockerfile`). Bump the pin after this release is tagged.

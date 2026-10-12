---
name: distress
description: >-
  Find and underwrite distressed / auction wholesale (assign-to-flipper)
  RealtyPad leads: Auction.com, HUD or open-data probate/foreclosure ingest,
  ARV, tax/lien gaps, max-offer stack, match_deal_buyers. Use for
  foreclosure/REO/auction sweeps, sell-to-flipper searches, or distress
  passes.
---

# RealtyPad — distressed auction / wholesale (assign-to-flipper)

Learned from the **2026-08-05 San Antonio Auction.com trial**. Prefer evidence; never invent tax or repairs.

## Hard rules

- **Never invent numbers.** Portal Cotality / AVM = **estimate** (store as sale comp `status=estimate`), not closed comps.
- `ask_price` = **starting bid or live current bid** (say which in notes). Re-fetch detail before bidding — bids move.
- **Reserve not met** + large bid increments means opening bid is not a clearing price.
- Annual tax ≠ delinquent / lien payoff. Leave both null until BCAD / tax collector / title packet verified.
- `wholesale_v1` spread = ARV − ask − repairs − **liens_owed** (null liens → 0). It still **ignores** assignment fee, flipper profit, and reserve reality — always run the **offer stack** in an agent comment.
- Aggressiveness: **aggressive** — pursue only clear asymmetric residual after the stack (≳ **$25–40k**).
- **Distress override on `ranked`:** keep `researching` until tax/title/repairs are credible, even if economics + `match_deal_buyers` already fit. Only then apply the buyer-fit gate → `ranked` (canonical gate otherwise: underwrite).
- **Distressed-sale listings, not only auctions.** On-market listings flagged **short sale**, **pre-foreclosure / foreclosure**, **REO / bank-owned** or **probate** (portal "Special listing conditions" grid → `properties.attributes`, or the write-up) get the same override. Automation surfaces them as `qa_flags.distress_conditions`. Record an **`action_needed` handoff** naming each condition and what a person must verify: short sale → lender approval of the price (the ask is not final) and timeline; pre-foreclosure / foreclosure → status, sale date, amount owed, liens; REO → bank terms and title; probate → court authority to sell, heirs, title. Say so in the brief — never present the ask as a firm price.

## Pipeline

```
Distress Progress:
- [ ] 1. Scope metro + channel (Auction.com first; HUD if thin)
- [ ] 2. Buyer prep: wholesale/fix_flip + roi_type=total_roi (cash_on_cash breaks wholesale match)
- [ ] 3. ensure_geo_markets for city/county chain
- [ ] 4. Apify pull → add_manual_leads (wholesale / auction_com|hud; shared defaults)
- [ ] 5. Triage cut (stale SALE_PENDING, non-resi, sale-day-tomorrow w/ no docs)
- [ ] 6. Research survivors: live bid, Cotality ARV, HOA, special notes, tax/liens
- [ ] 7. Offer brief comment + rescore
- [ ] 8. match_deal_buyers (snapshots on; link_matches sets scenario_run_id / auto-freezes live if none); status **researching** until tax/title/repairs credible, then buyer-fit → `ranked`
```

## Allowlisted connectors

Platform connectors for Auction.com / HUD / open-data distress — recipes in the data-source catalog ([data-sources.md](../ingest/references/data-sources.md)):

| Job | Detail page |
|-----|-------------|
| Auction.com | [auction-com.md](../ingest/references/data-sources/auction-com.md) |
| HUD / REO | [hud.md](../ingest/references/data-sources/hud.md) |
| Open-data distress | [probate-foreclosure.md](../ingest/references/data-sources/probate-foreclosure.md) |

Always set `maxTotalChargeUsd` (or `maxListings`). Do **not** call `apify/rag-web-browser` or any Store Actor outside the allowlist.

Normalize:

| Field | Value |
|-------|--------|
| `income_strategy` | `wholesale` |
| `lead_source_code` | `auction_com` or `hud` |
| `ask_price` | startingBid (or live current bid after detail refresh) |
| `auction_date` | Sale / auction day when known |
| `liens_owed` | Only when verified/estimated from tax collector / packet — never invent |
| `financing_profile` | `cash_offer` |
| notes | auction window, assetType, occupancy, reserve, Apify run id |

## Triage cuts (SA trial)

Drop or `watch`:

- `SALE_PENDING` / `SALE_PENDING_CLEARED` / stale auction dates (years old)
- Foreclosure trustee with **null** starting bid and no economics
- Non-residential / land (unless buyer book wants it)
- Opening bid already near Cotality est. (no spread)

Keep for research: active `AUCTION_IN_PROGRESS` with ask ≪ Cotality and residential SFR/PUD.

## Research pack

1. Re-open `source_url` — refresh **current bid**, reserve status, bid increment, HOA, special notes (structural, solar, occupied).
2. ARV: Cotality from listing → `arv` + `add_deal_comp(kind=sale, status=estimate)`. Prefer 3+ closed sale comps when available; mark secondary web comps as unverified.
3. `needs_rehab=true` when AS-IS / structural / unknown condition; leave `repair_estimate` null rather than inventing CapEx.
4. Tax: BCAD (`esearch.bcad.org`) + Bexar Tax Assessor for **annual** and **amounts owed**. If blocked, note the gap in a research **comment** (and/or leave tax fields null) — do not invent; do not use `kind=tax` observations for diary.
5. Liens: set `liens_owed` when tax collector / title packet is known; else note the lien gap in a **comment** (or `action_needed` handoff) until verified. Auction.com “liens may survive”; quitclaim risk.

## Offer stack (comment template)

```
ARV (comp/AVM, labeled)
− repairs (null → haircut in narrative only)
− flipper profit (~15–20% ARV or fixed)
− assignment fee
− known liens / buyer premium / HOA delinquencies
= max bid / max assignable offer
```

Compare to **live** ask/current bid, not the $1 opening when reserve is unmet.

## Buyer matching quirks

- Flipper/wholesale buyers need `roi_type=total_roi` (or no ROI gate). `cash_on_cash` / `hold_irr` / `cap_rate` fail wholesale (no monthly CF / NOI).
- Matcher treats wholesale `cash_needed` = **ask** (end-buyer funds). Low `cash_on_hand` near-misses high-ask auctions even when assignment fee is small — recruit higher-cash flippers or note the gap.
- Empty buyer book: if distress facts still block UW → stay `researching`/`blocked`; if economics clear and `scanned=0` → **`watch`** (do not soft-`passed`).

## Product gaps (ops notes)

- `liens_owed` feeds `wholesale_v1` spread (`ARV − ask − repairs − liens`). Null liens default to 0.
- `auction_date` is first-class; use `list_deals(sort=auction_date)` for sale-day queues.
- Offer stack still subtracts **flipper profit / assignment fee by hand** in the agent comment (not schema fields).
- BCAD esearch may fail SSL from some fetch paths — use browser or tax office when needed.

## After the pass

Brief: ingest counts, Apify run id/spend, survivors researched, match vs near-miss, what broke (tax access, live bid drift, cash gates). HUD fallback only if Auction.com returned &lt; ~5 usable actives.

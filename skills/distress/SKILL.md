---
name: distress
description: >-
  Find and underwrite distressed / auction wholesale (assign-to-flipper) leads:
  Auction.com or HUD ingest, ARV, tax/lien gaps, max-offer stack,
  match_deal_buyers. Use for foreclosure/REO/auction sweeps, sell-to-flipper
  searches, or distress passes.
---

# RealtyPad — distress

Call `get_agent_manual(workflow="distress")` before running this workflow. Follow that Markdown.

## Hard rules

- Never invent tax, title, or repairs. Stay `researching`/`blocked` until those are credible, then apply the buyer-fit gate. CapEx → `cost-estimate` skill.
- Ingest from the **detail page**, not the auction search card.
- Capture `auction_date`, `liens_owed`, starting bid vs ask, and listing URLs. Mark which price is which in notes.
- Strategy is usually `wholesale` or `fix_flip`. Financing default is `cash_offer`.
- After economics clear, `match_deal_buyers` decides `ranked` vs `passed` (empty book → `watch`). Hard external gaps → `blocked`.

## Flow

```
Distress Progress:
- [ ] Ingest (Auction.com / HUD / similar) via ingest skill
- [ ] Research tax/title/repairs/ARV
- [ ] Offer stack / max assign price (scenarios + underwrite)
- [ ] match_deal_buyers
- [ ] Status + UW comment
```

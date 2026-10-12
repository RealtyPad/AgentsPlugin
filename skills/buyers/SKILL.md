---
name: buyers
description: >-
  RealtyPad buyer engagement profiles, personalized share links, and
  buyer↔deal messaging. Use when checking unread share chat, reading what a
  buyer asked, drafting a reply, posting a workspace response on a share
  thread, matching deals to buyers, or managing buyer books (also called
  investors).
---

# Buyers / messaging

Buyer (`buyers`) rows are **engagement profiles**, not 1:1 with a person.
`buyer_type` is either:

| `buyer_type` | Meaning |
|--------------|---------|
| `investor` | Investment buyer — ROI / income-strategy criteria; matches `deal_purpose` investment or either |
| `individual` | Owner-occupant / primary buyer — no investment returns criteria; matches primary or either |

Only call a profile an **investor** when `buyer_type=investor`. Otherwise say **buyer**.

Personalized share links let a linked buyer mark interest and chat on a deal after
a magic-link sign-in (email or SMS to a contact on the profile). Workspace (and
agents) reply on the same thread.

**Never invent numbers** in replies. Do not invent offers, rents, ARV, taxes, or
legal/tax advice. Prefer citing deal fields / UW notes already on the record.

## When to use

- User asks to check messages, reply to a buyer, clear unread chat, or
  “what did the buyer say”
- After sharing a deal with a buyer and expecting questions
- Inbox sweep: `list_buyer_message_threads(unread_only=true)`

## Tools

| Tool | Role |
|------|------|
| `list_buyer_message_threads` | Workspace inbox; default `unread_only=true` |
| `list_deal_buyers` | Per-deal links + `unread_count` + share path |
| `list_deal_buyer_messages` | Full thread (chronological); marks read by default |
| `add_deal_buyer_message` | Post `author_kind=workspace` reply |
| `ensure_deal_buyer_share` | Create/refresh personalized share URL for a linked buyer |
| `ensure_buyer_share` | Create/refresh buyer portfolio URL (active deals, excl. passed); path `/share/investors/…` is legacy URL |
| `close_buyer` | Manually close an engagement profile (optional clone) |
| `reopen_buyer` | Reopen a closed engagement profile |
| `list_buyers` / `get_buyer` / `link_deal_buyer` / `match_deal_buyers` / `match_buyer_deals` | Buyer book; matching scores **live UW + selected scenario snapshots** and returns `scenario_run_id`. `match_buyer_deals` accepts `status` / `has_soft_gap` (same as `list_deals`; deprecated `needs_input` aliases) and optional `gate_ready_only` (researching + no soft gaps + scored). Each match / near miss has plain-language `reasons` / `gaps` (what people see) and structured `criteria` (`code`, `met`, `value`, `threshold` / `min` / `max`, `roi_type`, …) — read numbers from `criteria`, not by parsing the sentences |

## Engagement profiles

- Attach one or more contacts with roles (`primary`, `spouse`, `partner`, …)
- Contacts are reusable across profiles
- Profile `name` is editable; auto-generated until customized (`name_is_custom`)
- Lifecycle: `active` → manual `closed` via `close_buyer` (never auto-close on under_contract)
- Matching and default `list_buyers` scan **active** only
- **Hold resurfacing (automatic):** creating a buyer, reopening one, or changing any match criterion (strategies, ROI, cash, rate/profile, markets, price/beds, build type/builders, repairs cap, amenities, `buyer_type`) re-scans `watch` deals (Hold — no buyers yet) against that buyer. Each new fit gets one `action_needed` handoff on the deal (`raw.tag=watch_resurface`, `raw.buyer_id`, `raw.reasons`) and owners/admins get one digest email. The deal **stays** `watch`. When you see that handoff: re-run `match_deal_buyers(link_matches=true)` to confirm with live UW, set `ranked` via `update_deal_status` if it still fits, then delete the handoff. Notes/name/contact edits do not re-scan. An open handoff blocks a duplicate for the same deal × buyer; deleting it lets a later criteria change flag again.
- Set `buyer_type` correctly on create/update — it gates which `deal_purpose` values match
- Primary deals have **no income strategy**; do not treat them as LTR. Strategy prefs do not block owner-occ matches.
- Matching cash/rate: prefer thesis/snapshot blob (selected scenario `raw` / flat cell) or live `raw.cashflow` (`piti_v1` or `cf_v1`). If the blob is empty, use the deal’s **set** `financing_profile_id` (cash profile → 100% down, 0% rate). No deal profile → cash/rate unknown; do **not** default `traditional_good`. PITI is not monthly cashflow.
- **Thesis links:** `deal_buyers.scenario_run_id` ties a buyer to a saved UW snapshot. Prefer setting it on match/link so Overview Theses → Financials shows them under the right report. Messaging should cite that thesis’s numbers when they diverge from live.
- **Share freeze:** linking without a thesis auto-creates a draft scenario from live UW (`Share · {buyer}`) and stores it on the link. Personalized `/share/deals/{token}` pages read that snapshot (not live), including hold projections. Clearing the thesis re-freezes live rather than leaving the share floating.

`author_kind`: `investor` (guest on share — enum value is legacy) · `workspace`
(team/agent reply) · `system` (e.g. interest marked).

## Procedure — read and reply

1. **Find threads** — `list_buyer_message_threads(unread_only=true)` (or
   `list_deal_buyers` on a known deal and filter `unread_count > 0`).
2. **Load context** — `get_deal` (**core**; + `list_comments` if useful) for the deal; note
   ask/rent/status so the reply stays factual.
3. **Read thread** — `list_deal_buyer_messages(deal_id, buyer_id)` (leave
   `mark_read=true` unless peeking). Answer the **latest guest** questions;
   ignore near-miss matching noise unless asked.
4. **Draft** — short, professional, evidence-based. If data is missing, say so
   and offer to research — do not guess.
5. **Reply** — only when the user wants a send (or clearly asked you to reply):
   `add_deal_buyer_message(deal_id, buyer_id, body=…)`.
6. **Optional** — `add_comment` on the deal summarizing what was said / sent
   (author=`agent`) if the thread matters for later UW.

## Checklist

- [ ] Inbox or deal buyers shows the right `deal_id` / `buyer_id`
- [ ] Thread read before drafting
- [ ] Reply cites only verified deal facts (or explicitly marks unknowns)
- [ ] `add_deal_buyer_message` only after user intent to send (unless they said
      “reply for me”)
- [ ] No legal/tax advice; no invented numbers
- [ ] Prefer “buyer” in user-facing copy unless `buyer_type=investor`

## Share links (quick)

1. Buyer must be linked: `link_deal_buyer` / match → link.
2. Per-deal: `ensure_deal_buyer_share(deal_id, buyer_id)` → `url_path`
   (`/share/deals/…`; clipboard delivery in UI; agents return the path/token).
3. Portfolio: `ensure_buyer_share(buyer_id)` → `url_path` (`/share/investors/…`)
   listing active linked deals (excludes `passed`); cards deep-link to
   personalized deal shares (auto-ensured).
4. Generic (non-buyer) shares stay read-only — messaging requires a personalized
   deal share.

## Out of scope

- Email delivery of the share URL itself (clipboard / user paste for now). Buyers
  then sign in on that page via magic link to an email or phone already on the profile.
- Treating near-misses from `match_deal_buyers` as matches
- Renaming public `/share/investors` or `author_kind=investor` wire values (stable IDs)

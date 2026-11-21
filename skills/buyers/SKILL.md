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
- Inbox sweep: `buyer_message_list(unread_only=true)`

## Tools

| Tool | Role |
|------|------|
| `buyer_message_list` | No ids: the workspace inbox (default `unread_only=true`). With `deal_id` + `buyer_id`: the full thread, chronological; marks read unless `mark_read=false` |
| `buyer_list` | The buyer book (`deal_id` limits it to a deal's links, with `unread_count` and the share path; `status`, `q`) |
| `buyer_message_create` | `messages[{deal_id, buyer_id, body}]`: post `author_kind=workspace` replies (max 20) |
| `buyer_share` | `shares[]`: create/refresh the personalized share URL for a linked buyer (with `deal_id`), or the buyer portfolio URL (without; active deals, excl. passed; the path `/share/investors/…` is a legacy URL) |
| `buyer_close` | `buyer_ids[]`: manually close an engagement profile (`clone` optional); `reopen=true` reopens a closed one |
| `buyer_get` / `buyer_upsert` / `buyer_link` | Read one profile; create (no `buyer_id`) or patch (`buyer_id`) profiles; link a buyer to a deal with `links[]` or unlink with `unlink_buyer_ids[]` |
| `buyer_match` | Matching in both directions with `deal_id` (buyers for a deal) or `buyer_id` (deals for a buyer). It scores **live UW + selected scenario snapshots** and returns `scenario_run_id`. Filters: `deal_status`, `has_soft_gap`, `gate_ready_only` (researching + no soft gaps + scored), `scored_only`, `include_linked`, `link_matches`, `link_status`, `match_limit`, `near_miss_limit`. Each match / near miss has plain-language `reasons` / `gaps` (what people see) and structured `criteria` (`code`, `met`, `value`, `threshold` / `min` / `max`, `roi_type`, …) — read numbers from `criteria`, not by parsing the sentences |

## Engagement profiles

- Attach one or more contacts with roles (`primary`, `spouse`, `partner`, …)
- Contacts are reusable across profiles
- Profile `name` is editable; auto-generated until customized (`name_is_custom`)
- Lifecycle: `active` → manual `closed` via `buyer_close` (never auto-close on under_contract)
- Matching and default `buyer_list` scan **active** only
- **Hold resurfacing (automatic):** creating a buyer, reopening one, or changing any match criterion (strategies, ROI, cash, rate/profile, markets, price/beds, build type/builders, repairs cap, amenities, `buyer_type`) re-scans `watch` deals (Hold — no buyers yet) against that buyer. Each new fit gets one `action_needed` handoff on the deal (`raw.tag=watch_resurface`, `raw.buyer_id`, `raw.reasons`) and owners/admins get one digest email. The deal **stays** `watch`. When you see that handoff: re-run `buyer_match(deal_id, link_matches=true)` to confirm with live UW, set `ranked` with a status-only `deal_update` row if it still fits, then mark the handoff done (`observation_upsert` with `resolved=true`). Notes/name/contact edits do not re-scan. An open handoff blocks a duplicate for the same deal × buyer; deleting it lets a later criteria change flag again.
- Set `buyer_type` correctly on create/update — it gates which `deal_purpose` values match
- Primary deals have **no income strategy**; do not treat them as LTR. Strategy prefs do not block owner-occ matches.
- Matching cash/rate: prefer thesis/snapshot blob (selected scenario `raw` / flat cell) or live `raw.cashflow` (`piti_v1` or `cf_v1`). If the blob is empty, use the deal’s **set** `financing_profile_id` (cash profile → 100% down, 0% rate). No deal profile → cash/rate unknown; do **not** default `traditional_good`. PITI is not monthly cashflow.
- **Thesis links:** `deal_buyers.scenario_run_id` ties a buyer to a saved UW snapshot. Prefer setting it on match/link so Overview Theses → Financials shows them under the right report. Messaging should cite that thesis’s numbers when they diverge from live.
- **Share freeze:** linking without a thesis auto-creates a draft scenario from live UW (`Share · {buyer}`) and stores it on the link. Personalized `/share/deals/{token}` pages read that snapshot (not live), including hold projections. Clearing the thesis re-freezes live rather than leaving the share floating.

`author_kind`: `investor` (guest on share — enum value is legacy) · `workspace`
(team/agent reply) · `system` (e.g. interest marked).

## Procedure — read and reply

1. **Find threads** — `buyer_message_list(unread_only=true)` (or
   `buyer_list` on a known deal and filter `unread_count > 0`).
2. **Load context** — `deal_get` (**core**; + `comment_list` if useful) for the deal; note
   ask/rent/status so the reply stays factual.
3. **Read thread** — `buyer_message_list(deal_id, buyer_id)` (leave
   `mark_read=true` unless peeking). Answer the **latest guest** questions;
   ignore near-miss matching noise unless asked.
4. **Draft** — short, professional, evidence-based. If data is missing, say so
   and offer to research — do not guess.
5. **Reply** — only when the user wants a send (or clearly asked you to reply):
   `buyer_message_create(messages=[{deal_id, buyer_id, body}])`.
6. **Optional** — `comment_create` on the deal summarizing what was said / sent
   (author=`agent`) if the thread matters for later UW.

## Checklist

- [ ] Inbox or deal buyers shows the right `deal_id` / `buyer_id`
- [ ] Thread read before drafting
- [ ] Reply cites only verified deal facts (or explicitly marks unknowns)
- [ ] `buyer_message_create` only after user intent to send (unless they said
      “reply for me”)
- [ ] No legal/tax advice; no invented numbers
- [ ] Prefer “buyer” in user-facing copy unless `buyer_type=investor`

## Share links (quick)

1. Buyer must be linked: `buyer_link` / match → link.
2. Per-deal: `buyer_share(shares=[{deal_id, buyer_id}])` → `url_path`
   (`/share/deals/…`; clipboard delivery in UI; agents return the path/token).
3. Portfolio: `buyer_share(shares=[{buyer_id}])` → `url_path` (`/share/investors/…`)
   listing active linked deals (excludes `passed`); cards deep-link to
   personalized deal shares (auto-ensured).
4. Generic (non-buyer) shares stay read-only — messaging requires a personalized
   deal share.

## Out of scope

- Email delivery of the share URL itself (clipboard / user paste for now). Buyers
  then sign in on that page via magic link to an email or phone already on the profile.
- Treating near-misses from `buyer_match` as matches
- Renaming public `/share/investors` or `author_kind=investor` wire values (stable IDs)

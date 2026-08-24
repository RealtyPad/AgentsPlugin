---
name: buyers
description: >-
  Buyer engagement profiles, personalized share links, and buyer↔deal messaging.
  Use when checking unread share chat, reading what a buyer asked, drafting a
  reply, posting a workspace response on a share thread, matching deals to
  buyers, or managing buyer books. Prefer workflow=buyers (investors remains a
  legacy alias).
---

# RealtyPad — buyers

Call `get_agent_manual(workflow="buyers")` before running this workflow. Follow that Markdown.

## Hard rules

- Buyer rows are **engagement profiles**, not 1:1 with a person.
- Say “investor” only when `buyer_type=investor`. Matching respects `buyer_type` vs `deal_purpose`.
- Prefer `list_buyer_message_threads` for inbox (`list_investor_message_threads` is a legacy alias).
- `match_buyer_deals` accepts `status` / `needs_input` (same as `list_deals`) and optional `gate_ready_only` (researching + no soft CF gaps + scored).
- Thesis links: set `scenario_run_id` on match/link so shares freeze that snapshot. Do not invent financing or offer terms.

## Typical tools

`list_buyers`, `get_buyer`, `create_buyer`, `update_buyer`, `list_deal_buyers`, `link_deal_buyer`, `match_deal_buyers`, `match_buyer_deals`, `unlink_deal_buyer`, `ensure_deal_buyer_share`, `ensure_buyer_share`, `close_buyer`, `reopen_buyer`, `list_buyer_message_threads`, `list_deal_buyer_messages`, `add_deal_buyer_message`.

Underwrite / `advance_deal` use `match_deal_buyers` as the pursue/pass gate after economics clear.

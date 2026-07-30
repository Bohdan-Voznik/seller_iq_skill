# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

This is **not application code** — it's the content repository for the **SellerIQ Agent Skill**, an Agent Skills–format package (`agentskills.io`) that users install into Claude.ai/Claude Desktop or ChatGPT. Once installed, the skill lets the AI answer analytics questions over SellerIQ's Mixpanel data (Rozetka marketplace seller analytics: sales, unit economics, ad spend, SERP positions, stock, competitors) instead of one fixed report.

There is no build, lint, or test tooling here — the repo is markdown instructions, a JSON manifest, and two `.xlsx` templates. "Correctness" means the instructions produce correct agent behavior when interpreted by an LLM at runtime, not that anything compiles.

## Runtime architecture (read this before editing anything)

The skill does **not** ship its full ruleset inside the uploaded file. `skill/SKILL.md` is deliberately thin — frontmatter (`name`/`description` used for skill matching) plus a fixed procedure: fetch a manifest, then fetch only the reference files the current task needs, over HTTP from this repo's GitHub raw URLs at conversation time.

```
skill/SKILL.md (uploaded to Claude/ChatGPT)
   -> curl https://raw.githubusercontent.com/Bohdan-Voznik/seller_iq_skill/main/reference/index.json
        -> "version"/"updated" compared against what was already loaded this chat; on first
           touch or mismatch, re-fetch needed files instead of trusting cached content
        -> reference/core.md            always loaded first (hardblocks, cross-cutting traps)
        -> reference/<other>.md         loaded only if its "when" field in index.json matches
                                         the current task (e.g. ad questions don't pull stock docs)
```

**This means edits in this working tree have zero effect on any live skill session until committed and pushed to `main`** — `raw.githubusercontent.com` serves the `main` branch directly, there is no build/deploy/CDN step in between. A local-only change is invisible to the agent that reads it.

Consequences for making changes here:
- **Any content edit to a file listed in `reference/index.json` requires bumping `version` and `updated` in `index.json` in the same change.** The skill's own instructions (`SKILL.md` step 3) tell the agent to skip re-fetching if the manifest looks unchanged from earlier in the chat — an unbumped manifest means already-running sessions keep using stale rules.
- New reference files must be added to `index.json` with a `when` clause describing exactly which tasks should pull them — this is how conditional loading avoids dragging irrelevant docs into every request (`core.md` explicitly calls out "a question about ads shouldn't pull in stock docs").
- The manifest and reference docs are fetched with `curl`, deliberately not `web_fetch` — `SKILL.md` codifies this as a hard rule because `web_fetch` rejects URLs not already seen in that conversation's search/fetch history, which a fresh chat never has for this repo's URLs. Don't remove or weaken that instruction when editing `SKILL.md`.

## Reference file map (`reference/`)

| File | Loaded when | Content |
|---|---|---|
| `core.md` | always, first | Mandatory onboarding pipeline (project → business context → lookup tables), hardblocks, cross-cutting traps (status IDs aren't standard, ad spend not sliceable by sales channel, etc.), per-event gotchas |
| `connector-setup.md` | Mixpanel MCP connector itself is missing/erroring | Platform-specific connector setup instructions |
| `onboarding.md` | first connection, business-context survey, no events yet | Full onboarding steps 1–7 (core.md only has the condensed 1–3), user-facing progress text, business-context block format |
| `lookup-tables.md` | working with `products_crm`/`products_rozetka` lookup tables | Full schemas, 3-tier `products_crm` fill cascade, 5-tier `products_rozetka` source-priority cascade, xlsx verification workflow, Mixpanel lookup-table API quirks |
| `order-item-snapshot.md`, `product-position-snapshot.md`, `extra-product-position-snapshot.md`, `ad-item-snapshot.md`, `stock-snapshot.md` | task touches that event | Per-event field schemas and traps |
| `recipes.md` | user wants a known report pattern | Example cuts (retention, buyout rate, competitor comparisons, etc.) |
| `templates/products_crm-template.xlsx`, `templates/products_rozetka-template.xlsx` | building a verification draft | Pre-built xlsx with data validation + conditional formatting; filled starting at row 2, never rebuilt from scratch |

`SellerIQ skill spec.md` at the repo root is the original, much longer design spec `reference/*.md` was distilled from — treat `reference/` as the current source of truth; the spec is historical/background context, not something the runtime loads.

## Core domain model (needed to edit reference docs correctly)

Five Mixpanel events, each carrying `schema_v`:
- `order_item_snapshot` — one row per unit sold (`quantity` is always 1); `siq_removed=true` marks soft-deleted orders
- `product_position_snapshot` — own-product SERP tracking (`organic_global_pos`, `ad_global_pos`, `group`/`group_source`)
- `extra_product_position_snapshot` — full first page of SERP results (own + competitors) for a tracked search query
- `ad_item_snapshot` — ad campaign data; carries both `product_crm_id` and `rozetka_id`, the most reliable CRM↔Rozetka link
- `stock_snapshot` — stock level changes (`stock_change_type`: `out_of_stock`/`restock`/`stock_decrease_adjustment`/`sale`)

Two lookup tables reconcile identities across these events:
- **`products_crm`** (key `product_crm_id`) — links CRM products to Rozetka listings; base row set = every `product_crm_id` ever seen in `order_item_snapshot`
- **`products_rozetka`** (key `rozetka_id`) — classifies every Rozetka listing seen in position/stock events as own-product vs. competitor

Both are filled via deterministic priority cascades (documented in `lookup-tables.md`) that must run to completion in a single pass — a partial cascade presented as a finished draft is treated as a defect, not an acceptable shortcut (see "Наповнення products_crm: три ступені" in `lookup-tables.md` for the specific past failure mode this rule is guarding against).

Building/updating draft rows (reads + fuzzy matching) needs no user permission; **writing** to Mixpanel (`Create-Lookup-Table`, `upsert_rows`, `Update-Business-Context`) always requires explicit per-write confirmation, requested only once a concrete draft exists — never "can I start?" in advance.

## Two operating modes (relevant when editing instruction wording)

- **Interactive** — a human is present in the chat; the agent can ask clarifying questions, present button lists, and request write permission.
- **Automatic** — the user has set up their own recurring/scheduled invocation of the skill outside of this repo; no one is present to answer questions. The skill's job is only to have instructions that degrade gracefully here (build reports from whatever business-context/lookup data is already saved, never ask questions, state limitations directly in the report output, never perform any write). **This repo does not implement or need any scheduler/cron mechanism** — "testing automatic mode" means reviewing `SKILL.md`/`onboarding.md`/`core.md` wording for self-sufficiency without a chat partner, not building run infrastructure.

## Language convention

Content the skill itself generates and stores (business context entries, lookup-table columns, default user-facing copy) is written in Ukrainian. Reference docs follow the same convention. If replying to a user in another language, stored records still default to Ukrainian unless the user explicitly asks otherwise.

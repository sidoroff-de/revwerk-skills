---
name: context-foundation-inject
description: >
  Automatically pull relevant Context Foundation items into the conversation whenever marketing, sales,
  delivery, or RevOps topics come up — ICPs, personas, competitors, value proposition, brand story,
  positioning, pricing and packaging, sales motion, pipeline, channels, attribution, revenue metrics,
  CRM/tooling, campaigns, outreach, forecasting, lead handling, onboarding/activation, service delivery,
  retention and churn, expansion/upsell, cross-team handoffs and SLAs, and operating rules/policies
  (discounting, routing, escalation). Fires passively on topic match: when the user discusses, drafts,
  analyzes, or asks about anything in these areas, this skill checks the Foundation's registry and grounds
  the response in the company's own published context. Not invoked by an explicit command — it is background
  grounding, not a user-facing operation.
---

# Context Foundation Inject — client

This skill grounds ordinary marketing/sales/ops/delivery conversation in the Foundation's **published Context Items** (the full revenue lifecycle — pre- and post-sale, per the canonical taxonomy v3, ADR-0017: items are `(area, slug)`, grouped by the owning-function Areas `vision`/`marketing`/`sales`/`ops`/`delivery`). It is **read-only** and **passively triggered**: the user never asks for it by name; it fires because the topic matched. Design rationale: `client-skills/docs/adr/0001-client-skill-split-for-context-foundation-write-and-inject.md`.

## Prerequisites

- The **CF MCP** is connected (tools: `cf_registry_get`, `cf_item_get`). Any valid key role works — this skill never writes.

## Registry → item lookup

**`cf_registry_get()`** is an **index only** — metadata grouped by `area` (the five owning-function Areas), each entry carrying `slug`, `title`, `summary`, `triggers`, `published_at`. It does **not** include item bodies. Slugs may be sub-paths (`marketing/channels/seo`), and client-created custom items appear here alongside ratified ones — judge relevance the same way for both.

The registry response includes a top-level **`_docs`** string that states the fetch contract: each entry's `slug` is the key to read the full published item via **`cf_item_get(slug)`** — not an internal row id or UUID. Read `_docs` on every registry fetch; use the `slug` values it lists.

- **Correct:** `cf_item_get('icps')`, `cf_item_get('value-prop')`
- **Wrong:** `cf_item_get(area, slug)` (there is no `area` argument — `slug` alone is the whole call), a UUID, guessed slugs not in the registry

Full bodies come only from `cf_item_get(slug)` (`body_markdown` in the response). This lookup contract is owned by this skill (and `context-foundation-change` for writes) — not by `context-foundation-intake`.

## The loop (on every trigger)

1. **Browse, don't look up.** Call **`cf_registry_get()`** — note `_docs`, then scan Context Item metadata (`slug`, `title`, `summary`, `triggers`, grouped by `area`). Do **not** use `cf_items_by_tag_get(tag)`: there is no topic→tag mapping table, and inventing one silently misses items whose relevant tag isn't in the guessed mapping (e.g. `sales` is not a trigger tag at all).
2. **Judge relevance yourself**, from the title/summary/area/triggers metadata alone — before fetching any body. Relevant = the item's content would materially improve the response being written, not merely "same general subject."
3. **Drop already-surfaced items.** Exclude every slug you have already injected earlier in this conversation (see Session dedup below) — they're in context already.
4. **Fetch at most 5.** Read the full body of up to **5** of the most relevant remaining items via **`cf_item_get(slug)`**. Fewer is fine; five is the hard cap per trigger.
5. **Acknowledge, then use.** Before folding the content into the response, always show a one-line visible acknowledgment naming what was pulled in — e.g. *"Using your Foundation's ICP + Value Prop context here."* Never inject silently.
6. **Nothing relevant → do nothing.** If no not-yet-surfaced registry entry is judged relevant, fetch nothing, inject nothing, and show **no acknowledgment** — an empty pull is silent, and the response proceeds without Foundation content.

## Session dedup

Track (in conversation memory, not on disk) the set of Context Item slugs this skill has already fetched and announced in the **current conversation**. On a later trigger — same topic or a related one:

- Already-surfaced slugs are **not re-fetched, not re-announced, not re-injected**. Their content is already in the conversation.
- Newly relevant, not-yet-surfaced items **are** still injected, up to the same 5-per-trigger cap.
- If everything relevant has already been surfaced, this trigger is a no-op: no fetch, no acknowledgment (rule 6 above applies).

The tracking is per-conversation and dies with it — nothing persists across sessions.

## Scope: Context Items only

Judge relevance over `cf_registry_get()` and fetch only via `cf_item_get(slug)` for a registry-listed slug. Do **not** call `cf_item_get('goal')`, `cf_focus_list()`, or `cf_item_get('business-context')` as part of this skill — those are always-relevant background, a different concern from topic-triggered pulls, and other flows own them.

## Errors

This skill must never degrade the conversation it fires inside:

- **Connection / timeout** → retry the read once; if it persists, skip injection for this trigger and answer normally, with a one-line note that Foundation context couldn't be reached.
- **License / quota / expiry** → skip injection, note it in one line, and continue; don't turn a background grounding step into a blocking error.
- Never fabricate Foundation content when a read fails — an answer without context beats an answer with invented context.

## What this skill deliberately does NOT do

- **No explicit trigger.** "Inject my context" is not a command this skill answers to; it fires only on topic match.
- **No writes.** Creating/updating/deleting items or Focuses is `context-foundation-change`'s job.
- **No tag-mapping table.** Relevance is judged from registry metadata per trigger, never from a maintained topic→tag map.
- **No files.** Nothing is written to disk.


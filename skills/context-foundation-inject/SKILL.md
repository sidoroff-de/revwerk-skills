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

This skill grounds ordinary marketing/sales/ops/delivery conversation in the Foundation's **published Context Items** (the full revenue lifecycle — pre- and post-sale, per the canonical taxonomy v3, ADR-0017: items are `(area, slug)`, grouped by the owning-function Areas `vision`/`marketing`/`sales`/`ops`/`delivery`). It is **read-only** and fires on topic match, never by explicit command. Design rationale: `client-skills/docs/adr/0001-client-skill-split-for-context-foundation-write-and-inject.md`.

## Prerequisites

- The **CF MCP** is connected (tools: `cf_registry_get`, `cf_item_get`). Any valid key role works — this skill never writes Foundation data, and writes nothing to disk.

## Registry → item lookup

**`cf_registry_get()`** is an **index only** — metadata grouped by `area` (the five owning-function Areas), each entry carrying `slug`, `title`, `summary`, `triggers`, `published_at`. It does **not** include item bodies. Slugs may be sub-paths (`marketing/channels/seo`), and client-created custom items appear here alongside ratified ones — judge relevance the same way for both.

The registry response includes a top-level **`_docs`** string that states the fetch contract: each entry's `slug` is the key to read the full published item via **`cf_item_get(slug)`** — not an internal row id or UUID. Read `_docs` on every registry fetch; use the `slug` values it lists.

- **Correct:** `cf_item_get('icps')`, `cf_item_get('value-prop')`
- **Wrong:** `cf_item_get(area, slug)` (there is no `area` argument — `slug` alone is the whole call), a UUID, guessed slugs not in the registry

Full bodies come only from `cf_item_get(slug)` (`body_markdown` in the response). This lookup contract is owned by this skill (and `context-foundation-change` for writes) — not by `context-foundation-intake`.

## The loop (on every trigger)

1. **Browse, don't look up.** Call **`cf_registry_get()`** — note `_docs`, then scan Context Item metadata (`slug`, `title`, `summary`, `triggers`, grouped by `area`). Do **not** use `cf_items_by_tag_get(tag)`: there is no topic→tag mapping table, and inventing one silently misses items whose relevant tag isn't in the guessed mapping (e.g. `sales` is not a trigger tag at all).
2. **Judge relevance yourself**, from the title/summary/area/triggers metadata alone — before fetching any body. Relevant = the item's content would materially improve the response being written, not merely "same general subject."
3. **Drop already-surfaced items.** Track (in conversation memory, not on disk — nothing persists across sessions) the slugs already fetched and announced in the current conversation, and exclude them: they are never re-fetched, re-announced, or re-injected. Newly relevant items are still injected, up to the same cap.
4. **Fetch at most 5.** Read the full body of up to **5** of the most relevant remaining items via **`cf_item_get(slug)`**. Fewer is fine; five is the hard cap per trigger.
5. **Acknowledge, then use.** Before folding the content into the response, always show a one-line visible acknowledgment naming what was pulled in — e.g. *"Using your Foundation's ICP + Value Prop context here."* Never inject silently.
6. **Nothing relevant → do nothing.** If no not-yet-surfaced registry entry is judged relevant, fetch nothing, inject nothing, and show **no acknowledgment** — an empty pull is silent, and the response proceeds without Foundation content.

## Scope: Context Items only

Judge relevance over `cf_registry_get()` and fetch only via `cf_item_get(slug)` for a registry-listed slug. Do **not** call `cf_item_get('goal')`, `cf_focus_list()`, or `cf_item_get('business-context')` as part of this skill — those are always-relevant background, a different concern from topic-triggered pulls, and other flows own them.

## Errors

This skill must never degrade the conversation it fires inside:

- **Connection / timeout** → retry the read once; if it persists, skip injection for this trigger and answer normally, with a one-line note that Foundation context couldn't be reached.
- **License / quota / expiry** → skip injection, note it in one line, and continue; don't turn a background grounding step into a blocking error.
- Never fabricate Foundation content when a read fails — an answer without context beats an answer with invented context.


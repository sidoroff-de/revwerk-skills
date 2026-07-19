---
name: context-foundation-change
description: >
  Directly create, update, or delete Context Items and Focuses in the Context Foundation, outside an intake
  session — preview inline, confirm, then write via the CF MCP's direct CRUD tools. Triggers ONLY on an
  explicit human request: "add a context item", "save this to the foundation", "update our context",
  "change this context item", "remove this context item", "delete from the foundation", "create a focus",
  "archive this focus". Never self-invoked passively — if the conversation merely touches foundation content
  without an explicit change request, this skill does not fire. This skill is a THIN CLIENT of the hosted CF
  MCP: it validates against server-provided vocabularies and writes nothing to disk.
---

# Context Foundation Change — client

This skill makes **direct, human-requested writes** to a Context Foundation: create/update/delete a Context Item (`cf_item_create` / `cf_item_update` / `cf_item_delete`), and create/archive a Focus (`cf_focus_create` / `cf_focus_archive`). It bypasses intake/synthesis entirely — what the user confirms is what gets written, verbatim. Design rationale: `client-skills/docs/adr/0001-client-skill-split-for-context-foundation-write-and-inject.md`.

**One operation per invocation.** A user adding several items invokes the skill repeatedly, each with its own preview/confirm cycle — no multi-item loop.

## Prerequisites (check first)

- The **CF MCP** is connected (tools: `whoami`, `cf_item_create`, `cf_item_update`, `cf_item_delete`, `cf_focus_create`, `cf_focus_archive`, `cf_vocabulary_get`, `cf_item_get`, `cf_registry_get`, `cf_focus_list`).
- A **license key** with a `contributor`-or-higher role is configured in the MCP connector — every write tool here rejects a reader-only key.
- Nothing is written to disk by this skill. Previews are inline conversation markdown, never a file.

## Preflight

Call **`whoami`** before asking the user anything. Silent on success. On failure, classify it with the error buckets below (no separate "not connected" category) and stop before any question is asked.

## Vocabularies (fetch, never hardcode)

Before offering a Create or an Update that touches `area`, `injection_mode`, or `triggers`, call **`cf_vocabulary_get()`**. It returns `{ areas, injection_modes, triggers }`:

- `areas` — the **owning-function Areas** (canonical taxonomy v3, ADR-0017): `vision`, `marketing`, `sales`, `ops`, `delivery`, plus the reserved sentinel `"other"` (flagged exception for content fitting no Area) and `"system"` (reserved for `system/config` — never propose it for domain content). The item's `area` field stores one of these values — the function that **authors and owns** the content, never who consumes it. Each Area lists its **ratified slugs**, each with a one-line classification `hint` (e.g. `pipeline` — "Deal stages, entry/exit criteria, and the disqualifier gate on the path to close") — use hints to **propose** an `(area, slug)` placement from the content, not just to validate a value the user already typed. There are no clusters and no categories — those terms are retired. See "Classifying `area` and `slug`" below.
- `injection_modes` — the valid `injection_mode` values.
- `triggers` — the controlled trigger-tag vocabulary.

Use these to validate the user's values (and to offer a pick-list when they're unsure) **before** the tool call — catch an invalid value client-side instead of leaving it to server-side rejection alone. Never rely on a remembered copy of these lists: they have changed server-side before and will again; the live tool response is the only source of truth.

## Classifying `area` and `slug`

Don't just ask the user to pick off a bare list. Read the item's `title` and `body_markdown` (or, for an Update, the changed content) against the ratified slugs' `hint`s from `cf_vocabulary_get()`, and **propose** a placement as part of the preview — the user confirms or overrides it like any other previewed field, same mechanism, no extra step. A placement is the owning **Area** plus a **slug**, which is one of:

- a **ratified slug** — the content *is* that item (e.g. `marketing/channels`);
- a **sub-path under a ratified slug** — the content refines it (e.g. `marketing/channels/seo`; the parent stays the index + general info);
- a **custom slug** anywhere below the Area — custom items are ordinary direct writes, unknown to intake and routing, requiring no ratification;
- for an operating rule: the owning Area's reserved **`rules/` namespace** (e.g. `sales/rules/discount-floor`).

`"other"` is a **flagged exception, never a silent default**. Only propose it when no Area genuinely owns the content, and say so explicitly in the preview (e.g. "No Area fits this content — save under `other`?") so the user is confirming that specifically, not waving through a default they didn't notice.

## Create flow (Context Item)

On an explicit request to add a new Context Item:

1. Call `cf_vocabulary_get()`.
2. Collect from the user: `slug`, `title`, `injection_mode`, `triggers`, `summary`, `body_markdown`. For `area` and `slug`, propose a best-fit placement per "Classifying `area` and `slug`" above instead of asking the user to supply them blind — they can still override. Validate `area` (one of the Areas + `"other"`), `injection_mode`, and every `triggers` tag against the fetched vocabularies; `slug` is free-form (a ratified slug, a sub-path, or a custom slug). On a miss, show the valid options and re-collect that field.
3. Render the complete proposed item as a **markdown preview inline in the conversation** — every field including the proposed `area`, the full `body_markdown`, nothing truncated, nothing written to disk.
4. **Only after the user explicitly confirms**, call `cf_item_create` with the confirmed fields.
5. Relay the result: success ("published as `<slug>`"), or the server's rejection message **verbatim** (e.g. a duplicate slug) — never paraphrased.

## Update flow (Context Item)

On an explicit request to change an existing Context Item:

1. Call **`cf_item_get(slug)`** for the item's current published state. If the call fails (no such published item), relay that verbatim and stop.
2. Collect **only the fields the user wants to change**. If `area` is among them (or the content being changed makes the current placement a poor fit), propose a best-fit placement per "Classifying `area` and `slug`" above rather than asking the user to supply it blind. If `injection_mode` or `triggers` is among the changed fields, validate the new value against `cf_vocabulary_get()`, same as Create.
3. Render a **before/after diff-style preview**: for each changed field only, the current value alongside the proposed value. Unchanged fields are not shown and not sent.
4. **Only after the user confirms the diff**, call `cf_item_update` with the item's `slug` plus just the changed fields (PATCH-style — unmentioned fields must not appear in the call).
5. Renaming is supported: pass the new slug as `new_slug`. Do **not** attempt to find or repair any Focus whose `body_markdown` references the old slug — per ADR-0001 that's left as-is (Focus links resolve by id, not slug).
6. Relay a rejection (e.g. `slug` doesn't resolve to an existing item) **verbatim**, not paraphrased.

## Delete flow (Context Item)

Deletion archives the item server-side, but **no undelete is exposed to this client — treat it as permanent**. That's why Delete alone requires **two** confirmations, distinct from the single confirmation Create/Update use:

1. Call **`cf_item_get(slug)`** and show the user exactly which item will be archived — at minimum its `slug` and `title`.
2. **First confirmation:** confirm the action itself ("Archive `<slug>` — `<title>`?").
3. **Second confirmation:** a separate, explicit confirmation that states plainly this **cannot be undone from here** (e.g. "This can't be undone from this client — there is no restore. Really archive it?"). Do not merge the two into one question.
4. Only after **both** confirmations, call `cf_item_delete` with the `slug`.
5. Relay a rejection (e.g. slug already archived or never existed) **verbatim**.

## Focus flows

These deliberately **duplicate** the Focus-handling logic in `context-foundation-intake`'s start-of-session checks (ADR-0001) — that skill is not touched, referenced, or delegated to here.

**Create a Focus:**

1. Collect `body_markdown` — a short description of the standing initiative/theme.
2. Preview it inline as markdown.
3. After the user confirms, call `cf_focus_create({body_markdown})` and relay the returned `focus_id`.

**Archive a Focus:**

1. Identify the target `focus_id`. If the user doesn't already know it, call **`cf_focus_list()`** and present the active Focuses (`id` plus a short excerpt of each `body_markdown`) for them to pick from.
2. Preview the archive action (the chosen `id` + excerpt).
3. After the user confirms, call `cf_focus_archive({focus_id})` and relay the result.

A reader-role key's rejection from either tool is a **License/quota/role** error (bucket 1 below) — stop and tell the user their key needs a higher role; a retry or reworded input won't fix it. Same bucket as the Item flows.

## Errors

Apply these three buckets uniformly to every tool call, including the `whoami` preflight — the same buckets as `context-foundation-intake`:

1. **License / quota / expiry / role** → stop, tell the user to contact the provider (or that their key needs a contributor/approver role). Do not retry.
2. **Connection / timeout** → retry once; if it persists, report and stop. Do not fabricate a result.
3. **Rejected input** (e.g. duplicate slug, unknown slug, trigger tag outside the vocabulary that slipped past client-side validation) → surface the server's rejection message **verbatim**, ask the user to revise, and resubmit.

## What this skill deliberately does NOT do

- **No passive triggering.** It acts only on an explicit human change request — topic-triggered context reads are `context-foundation-inject`'s job.
- **No synthesis.** The user's confirmed text is written as-is; if they want help drafting `body_markdown`, that drafting happens in conversation before the preview, and the previewed text is still what's sent.
- **No files.** Previews live in the conversation; nothing touches disk.
- **No link repair.** Renamed slugs may leave stale item cross-references (the link markers Focus bodies embed to point at a slug) pointing at the old slug — leave them.
- **No intake.** Guided, question-driven population of the Foundation is `context-foundation-intake`'s job.


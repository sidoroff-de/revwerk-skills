---
name: context-foundation-intake
description: >
  Run a Context Foundation intake — a guided session that turns how your business runs into a populated,
  machine-readable Context Foundation that AI tools can query. Triggers on: "context foundation intake",
  "foundation questionnaire", "discovery workshop", "gather our context", "kick off a context foundation",
  "onboard our context", "enrich the context foundation". This skill is a THIN CLIENT: it drives the hosted
  CF-Intake MCP and relays what it returns. It contains no method of its own and writes nothing to disk — the
  questions, synthesis, and storage all live behind the MCP.
---

# Context Foundation Intake — client

This skill **orchestrates** a Context Foundation intake. The actual method (what to ask, how to shape each draft) lives in the hosted **CF-Intake MCP**, not here. Your job is to be a faithful **relay**: pass the user's answers to the MCP, and relay what it returns — synthesized text verbatim, questions unedited — back to the user.

> **MVP note (issue #40).** A sufficient answer now **publishes immediately** — there is no pending-approval state. `cf_intake_approve_draft` still exists on the server but this skill never calls it. Relay `draft_preview` as an **FYI** ("here's what was just published"), not a request to approve — if the user wants changes, that's a **follow-up answer that publishes a new revision on top** of what's already live, not an "undo" of the last publish.

## Prerequisites (check first)

- The **CF-Intake MCP** is connected (tools: `whoami`, `cf_intake_start`, `cf_intake_submit_answers`, `cf_focus_create`, `cf_focus_archive`, `cf_focus_list`, `cf_item_get`, `cf_session_get`).
- A **license key** already scoped to the right Foundation and role is configured in the MCP connector. Foundation creation (`cf_admin_init_foundation`) is a separate, practitioner-only admin step — out of scope here.
- Nothing is written to disk by this skill. There is no target folder/repo to set up.

## Preflight

Call **`whoami`** before asking the user anything. Silent on success. On failure, classify it with the error buckets below (no separate "not connected" category) and stop before any question is asked.

## Start-of-session checks (before asking the user anything)

0. If you already have a `session_id` from earlier in this conversation, or the user gives you one to continue an interrupted intake, this is a **resume** — go to **Resume** below instead of the checks in this section. `focus_id` is bound to the session at creation and never re-selected on resume, so none of steps 1–3 below apply in that case. Otherwise, this is a cold start; continue with steps 1–3 here.
1. Call **`cf_focus_list()`** to list the Foundation's active Focuses (each has an `id` and `body_markdown`).
   - **One or more exist** → present them to the user (`id` plus a short excerpt of `body_markdown` for each) and have them pick the one that matches what they're working on — or confirm it, if there's exactly one. Capture the chosen Focus's `id` as `focus_id`.
   - **None exist, or none fit** what the user describes → call **`cf_focus_create({body_markdown})`** with a short description of the initiative, then use the returned `focus_id`. Same single gate either way — a brand-new Foundation with zero Focuses is just the empty-list case of this step, not a separate first-session bootstrap flow.
   - `cf_focus_create`/`cf_focus_archive` require a contributor/approver-role key. If creation fails because the connected key is reader-only, treat it as the **License/quota** error bucket below (stop, tell the user their key needs a higher role) — not something a retry or reworded `body_markdown` fixes.
2. Call **`cf_item_get('goal')`** — `'goal'` is a fixed literal slug, never derived from the registry or from `area`.
   - **Succeeds** → the Foundation already has a Business Goal published (its `body_markdown` is the SMART statement). Skip collecting `goal` entirely — you'll call `cf_intake_start` without it and go straight to the area loop.
   - **Fails with a not-found error** (e.g. "no published context item 'goal'") → no Goal has been published yet; you'll collect `goal` from the user in the loop below.
   - **Fails any other way** (connection/timeout/license) — fail open: tell the user in one line ("Couldn't check whether a goal is already set (connection issue) — I'll ask you directly."), then proceed as if no Goal exists. This is a UX shortcut only — it must never block the session.
3. Call **`cf_item_get('business-context')`** — `'business-context'` is a fixed literal slug, never derived.
   - **Succeeds** → skip drafting/asking about the business description. Send `business_context: ""` to `cf_intake_start`; the server's own fallback fills it in from the stored value.
   - **Fails with a not-found error** (e.g. "no published context item 'business-context'") → draft 2–3 sentences from material already at hand in this session (project files, docs, discovery-call notes), show the draft to the user for confirmation/correction, and send the confirmed text as `business_context`.
   - **Fails any other way** — same fail-open rule: report it in one line, then fall back to draft-and-confirm as if no Business Context exists.

## Inputs to gather from the user

- **focus_id** — selected (or newly created) in start-of-session check 1, before anything else here is collected. Required on every `cf_intake_start` call.
- **intent** — one line on what *this session* is about, why you're doing the intake now. Free text that dies with the session.
- **goal** — only if `cf_item_get('goal')` failed with a not-found error. The company's durable 6–12 month commercial objective, one line.
- **business_context** — only if `cf_item_get('business-context')` failed with a not-found error. See the draft-and-confirm procedure above.

**Focus vs. Intent — easy to conflate, keep them distinct:**
- **Focus** is the standing, persisted theme that survives across sessions (e.g. "Automate daily routine to improve efficiency"). Selected once, up front, from `cf_focus_list()`.
- **Intent** is this session's one-line, ephemeral task (e.g. "Fix existing n8n pipeline"). Typed fresh every session and never persisted beyond it.

## The loop (fixed — do not improvise)

**Naming note (taxonomy v3, ADR-0017):** the wire field `area_id` carries the item's ratified **slug** (`icps`, `pipeline`, …); the server owns the slug→Area mapping and the weighting. Relay `area_id` values exactly as received — never translate or invent them.

**Accumulation rule:** the MCP is stateless — on a `"followup"` resubmit, `raw_answers` must be the *full* running answer for that `area_id` so far (prior rounds + the new text), not just the latest fragment, since the server keeps no memory between calls. Reset it when the area publishes. Before every submit, show the user the exact `raw_answers` text about to be sent and let them confirm or edit it.

1. Call **`cf_intake_start`** with `{intent, focus_id, goal, business_context}`, omitting/emptying `goal` and `business_context` per the start-of-session checks above. `focus_id` is always the id captured in start-of-session check 1 — never omit it.
   - If this key already has another active session (e.g. a repeat call), the server silently reuses that session with its originally-bound `focus_id`/`intent` — the `focus_id` you pass here is ignored in that case, the same as a differing `intent` already was.
   - `status: "goal_followup"` → present `questions` to the user, collect an answer, call `cf_intake_submit_answers` with `{session_id, area_id: "goal", raw_answers}`. While that response's `status` is `"followup"`, present its `questions` and resubmit. Once it's `"published"` — the goal just published — go to **Publish FYI** below (using that response's `draft_preview`/`weighting`), then continue to **Area questions** with its `next`.
   - `status: "areas_active"` → the goal is now published (either it already was, or this call just published it). If `draft_preview` is set, go to **Publish FYI** below for the goal, then continue to **Area questions** with `next`. If `draft_preview` is absent, the goal was already published in an earlier session — go straight to **Area questions** with `next`.
2. **Publish FYI (goal or area):** present `draft_preview` to the user **verbatim, in full** — never summarize or truncate it — framed as already-live content, not a pending decision (e.g. "Here's what was just published for `<area>`:"). If the goal was just published, also relay `weighting` as a one-line FYI (e.g. "we'll go deep on X, lighter on Y"); it's informational, not a question requiring a response.
   - If the user wants changes, collect their revised/additional answer text and call `cf_intake_submit_answers` again with the same `session_id`/`area_id` — this publishes a **new revision on top** of what's live, it does not undo the previous publish. Never edit the synthesized text yourself — you have no synthesis capability; only the MCP does.
   - Otherwise, continue the loop: `next: null` → the session is complete, tell the user and stop; `next` set → go to **Area questions** with `next`.
3. **Area questions:** present `next.questions` for `next.area_id`, collect a free-text answer, call `cf_intake_submit_answers` with `{session_id, area_id, raw_answers}`.
   - `status: "followup"` → present `questions` again, collect, resubmit.
   - `status: "published"` → go to **Publish FYI** above.
4. The loop ends when a `cf_intake_submit_answers` response's `next` is `null`. There is no completeness/gaps/golden-question report — the session just ends.

## Resume

Before asking the user anything, call **`cf_session_get(session_id)`** for the `session_id` you're resuming. `focus_id` is bound to the session at creation and immutable — it is **never re-prompted here**. The response doesn't surface `focus_id` at all (its fields are `status`, `goal_published`, `areas`, `pending_area_id`, `pending_draft_text`, `next` — nothing else), which is fine: `cf_intake_start` is never re-called on resume, so there's nothing to check or re-supply.

- **`status` is anything other than `"active"`** (`"completed"`, `"abandoned"`, or any other value the tool ever reports) → tell the user plainly that this session can't be continued and they need to start a new one. No per-status special messaging — treat every non-`"active"` status identically, then fall back to a cold start (a fresh `cf_intake_start` call) if the user wants to proceed.
- **`status: "active"`** → branch on the rest of the payload, in this order:
  1. **`pending_area_id` is set** (an open, unapproved draft exists — its text is `pending_draft_text`). Under the #40 MVP bypass this should essentially never happen (a sufficient answer publishes immediately, so nothing is left pending) — treat it as a legacy/rare case, not the normal resume path. If you do see it, present `pending_draft_text` verbatim as an FYI for `pending_area_id`, then continue with a follow-up `cf_intake_submit_answers({session_id, area_id: pending_area_id, raw_answers})` if the user wants changes.
  2. **No `pending_area_id`, and `goal_published` is false** → no open draft, and the goal was never anchored for this session. Do not try to reconstruct the prior back-and-forth (raw answers aren't persisted server-side) — fall back to the same goal-collection behavior as a cold start: ask the user for their goal directly, then call `cf_intake_submit_answers({session_id, area_id: "goal", raw_answers})` (the session already exists, so this continues it — do not call `cf_intake_start` again). Handle that response exactly like the goal-followup loop in **The loop**, step 1: keep resubmitting while `status` is `"followup"`; once `"published"`, go to **Publish FYI**, then **Area questions** with `next`.
  3. **No `pending_area_id`, `goal_published` is true, and `next` is set** → present `next.questions` for `next.area_id`, exactly like **Area questions** above, and continue the loop with `cf_intake_submit_answers({session_id, area_id: next.area_id, raw_answers})`.
  4. **No `pending_area_id` and `next` is null** → nothing left to do; tell the user the session is already complete.

Every regenerated question or draft here is fresh, not reconstructed — the MCP has no memory of the prior conversation's raw answers either, only what it already persisted (the open draft's stored text, or a newly synthesized opening question for the plan's current area). Once resumed, the rest of the loop (Publish FYI, Area questions, Errors) behaves identically to the cold-start path.

## Relay discipline (the important part)

- **Do not invent, reorder, or skip questions.** Present only what the MCP returns.
- **Do not answer on the user's behalf** or infer answers. "We don't track this yet" is a valid answer — pass it through.
- **`draft_preview` is the full synthesized text, not a summary** — relay it in full, every time. It's already published by the time you see it (issue #40) — present it as an FYI, not a pending decision.
- **Pass answers as raw text.** Don't pre-summarize; the MCP judges sufficiency and does the synthesis.
- **Nothing is written to disk.** There are no chunks, no files, no paths. Export is a separate, deferred concern outside this skill.

## Errors

Apply these three buckets uniformly to every tool call in the loop, including the `whoami` preflight:

1. **License / quota / expiry** → stop the session, tell the user to contact the provider. Do not retry.
2. **Connection / timeout** → retry once; if it persists, report and stop. Do not fabricate questions or content.
3. **Rejected input** (e.g. `cf_intake_submit_answers` rejects the payload as insufficient or too long) → surface the server's rejection message verbatim, ask the user to revise, and resubmit. No retry cap, and no client-side pre-validation of things like character limits — the server is the sole source of truth for validity.

## What this skill deliberately does NOT contain

No Area Map, no question bank, no per-area rules, no synthesis logic, no file-writing logic. If you find yourself reasoning about *which* question to ask next, *how* to shape a draft, or *where* to save something, stop — none of that is this skill's job. Ask the MCP.


---
name: context-foundation-resolve
description: >
  Passively resolves cf://... reference strings (cf://items/{slug}, cf://tags/{tag}, cf://registry,
  cf://goal, cf://business-context, cf://_config, cf://vocabulary, cf://search/{query},
  cf://sessions/{session_id}) into the matching CF MCP tool call whenever one appears in the
  conversation — typed, pasted, or embedded in fetched content. Not invoked by an explicit command.
---

# Context Foundation Resolve — client

`cf://...` strings are a content-reference convention, not real URLs — fetching one as a web request
always fails. Whenever one appears (user-typed, pasted, or embedded in a Context Item's
`body_markdown` or a search snippet already fetched), resolve it via the matching tool below.

## Mapping

`slug`/`tag`/`query`/`session_id` come from the URI exactly as written — never invented.

| Reference | Tool call |
|---|---|
| `cf://items/{slug}` | `cf_item_get(slug)` |
| `cf://goal` | `cf_item_get('goal')` |
| `cf://business-context` | `cf_item_get('business-context')` |
| `cf://_config` | `cf_item_get('_config')` |
| `cf://tags/{tag}` | `cf_items_by_tag_get(tag)` |
| `cf://registry` | `cf_registry_get()` |
| `cf://vocabulary` | `cf_vocabulary_get()` |
| `cf://search/{query}` | `cf_search(query)` |
| `cf://sessions/{session_id}` | `cf_session_get(session_id)` |

`cf://focus` / `cf_focus_list()` is excluded — the server already inlines linked items' content into
a Focus, so there's nothing left to resolve there.

## Rules

- **One level, no auto-recursion.** Resolve references in the current turn's material once. If
  resolved content contains further `cf://...` references, name them as available but don't
  auto-fetch — resolve only if asked next.
- **Acknowledge, then use.** One line naming what was resolved (e.g. *"Resolved cf://items/personas."*)
  before using the content. Never resolve silently.
- **Not found / connection issue** → one-line note, don't fabricate, continue without it.

## Out of scope

Topic-triggered pulls (`context-foundation-inject`'s job) and writes (`context-foundation-change`'s
job). This skill only reacts to a reference already in front of it.
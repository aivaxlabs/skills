---
name: aivax-memories
description: Operate AIVAX long-term user memories — the `Remember` built-in tool (`memory_save`, `memory_search`) and the reserved `@memories` RAG collection that stores them. Load when the agent must enable memory for a gateway or request, audit or delete a user's memories, or replace calls to the removed `/api/v1/memories` endpoints.
---

# Memories

Long-term memories are documents in the account's reserved `@memories` RAG collection, keyed by the end user's external user ID. The model writes and reads them through the `Remember` built-in tool. They survive across sessions and are not conversation history.

The dedicated `/api/v1/memories*` endpoints (list, get, migration prompt, delete) no longer exist, and neither do the `DateReminder` format and the `Calendar` built-in tool. Legacy memories were migrated into `@memories` as plain-text documents (reminders included), and the Calendar flag was cleared from existing gateways and batch workflows. Manage memories through the RAG endpoints below.

## When To Use

Use this sub-skill to enable memory on a request or gateway, explain how memories behave, or audit, export, and delete a user's memories.

Do not use it for:

- Application knowledge bases. Use `references/rag/` instead.
- Sandbox files. Use `references/shell-io/` instead.
- Scheduled follow-ups for web chat clients. Use `references/hook-schedules/` instead.

## Enable Memory

- Chat completions: include `Remember` in `builtin_tools.tools`.
- Gateways: enable `Remember` in `parameters.sentinelOptions.enabledFunctions`. `parameters.builtinFunctionsOptions.allowSharedMemory` (default `true`) lets the gateway read memories created by other gateways; when `false`, it only sees memories it created. Migrated legacy memories carry no gateway link, so a gateway with `allowSharedMemory: false` does not see them.
- Memory needs a user identity: send the end user's ID (the chat-completions `user` field, or the session external user ID). Without it the tools fail with "Memories are unavailable".

## Tools

- `memory_save`: omit `id` to create; `id` plus `content` replaces; `id` plus `expiresInDays` without `content` changes only the expiration; `id` alone deletes. Expiration is optional; expired memories stop appearing in search and are removed automatically.
- `memory_search`: provide exactly one of `query` (semantic) or `filter` (a document filter, newest first). Memories are never preloaded, so the model must search. Results carry the memory ID, content, and its dates.

Memories are RAG documents: a new or changed memory is searchable only after indexing finishes (longer under backlog), and writes and searches use the RAG embedding allowance or billing, RAG rate limits, and require a positive balance (402 otherwise). Do not store secrets or sensitive personal data.

## Layout (For Auditing)

- Collection: `@memories` (find its ID with `GET /api/v1/collections`).
- Document `id` is the memory ID; `name` is `<externalUserId>/<uuid-without-hyphens>`.
- Metadata: `external_user_id`, `conversation_token`, `gateway_id`, and `expires_at` (Unix seconds; null means no expiry).

## Inspect And Delete

```text
GET    /api/v1/collections
GET    /api/v1/collections/<id>/documents?filter=<externalUserId>
POST   /api/v2/query/browse   { "collections": ["<id>"], "filter": "metadata.external_user_id = \"<externalUserId>\"" }
GET    /api/v1/collections/<id>/documents/<memory-id>
DELETE /api/v1/collections/<id>/documents/<memory-id>
```

- A filter-only `/query` returns the user's memories newest first. Filter text comparisons ignore case and accents; confirm `metadata.external_user_id` in each result before acting.
- There is no bulk delete: list the user's memories, then delete them one by one.
- Names starting with `@` are reserved: creating or renaming a collection to one returns 403, and editing or generating context for an existing one returns 403. Document upsert and import are not blocked on `@memories`, but avoid hand-writing memories: a document without the metadata above is never returned by `memory_search`.
- The code does not block deleting or resetting `@memories` itself, which would erase every user's memories in the account. Treat it as a destructive action and never do it unprompted.

## Safety

- Deleting memories is irreversible. Every delete requires explicit user approval; state the exact external user ID and memory IDs before calling.
- `GET /api/v1/collections/<id>/export.jsonl` exports `docid`, `text`, `reference`, and `tags` only, so metadata (owner, expiry, gateway) is not preserved. It is a content backup, not a restorable one.
- Memory contents are personal data. Never log them or forward them outside the destination the user named.

## Validation

- Before deleting, confirm each target belongs to the expected external user.
- After deleting, confirm the memory no longer appears in a fresh filtered query or browse.
- After enabling memory, confirm the request carries a user ID and that a saved memory is found by `memory_search` once indexed.
- The trace ID is preserved.

## Escalation

- Memories are saved but not found: check that indexing finished (`state=queued` in the document listing), the user ID matches, and `allowSharedMemory` fits the gateway.
- A call to `/api/v1/memories` fails: the endpoint was removed; use the flows above.
- The delete target is ambiguous: stop, re-list, and ask the user to confirm the exact memory.
- Embedding cost or indexing questions: load `references/rag/`.

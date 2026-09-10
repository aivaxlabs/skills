---
name: aivax-memories
description: List, read, migrate, and delete persisted user memories keyed by external user ID on AIVAX. Load when the agent must inspect or manage long-lived persisted memories (Text or DateReminder formats) for an external user.
---

# Memories

This sub-skill owns persisted user memories. A persisted memory is a long-lived record keyed by `externalUserId` with a format of `Text` or `DateReminder`, optionally linked to an AI gateway. Persisted memories are not conversation memory; they survive across sessions for the given external user.

All routes share the prefix `/api/v1/memories`.

## Discovery Workflow

```text
GET /api/v1/memories?filter=<text>&format=<Text|DateReminder>&gatewayId=<uuid|shared>
GET /api/v1/memories/<id>
GET /api/v1/memories/generate/migration?externalUserId=<id>
DELETE /api/v1/memories?id=<uuid>
DELETE /api/v1/memories?externalUserId=<id>&format=<Text|DateReminder>
```

## When To Use

Use this sub-skill when the agent must list persisted memories (optionally filtered by text, format, or gateway), read one memory in full, build a migration prompt carrying an external user's persisted context, or delete persisted memories.

Do not use this sub-skill for:

- Conversation memory inside a gateway session. That belongs to gateway memory options; load `references/ai-gateways/` instead.
- Sandbox files. Use `references/shell-io/` instead.

## Endpoints

- **List Memories** (`GET /api/v1/memories`): optional `filter` narrows by memory id, content, or external user id; optional `format` narrows to `Text` or `DateReminder`; optional `gatewayId` narrows to one gateway UUID or `shared` for memories with no gateway link. List items carry a content preview truncated to 120 characters.
- **Get Memory** (`GET /api/v1/memories/<id>`): returns one memory with full content, external user id, format, creation and expiration times, and linked gateway info. Returns 404 when the id does not exist for the authenticated account.
- **Generate Memory Migration Prompt** (`GET /api/v1/memories/generate/migration?externalUserId=`): builds a prompt containing the external user's persisted memories plus scheduled and recurring message context for migration. Requires `externalUserId`.
- **Delete Memories** (`DELETE /api/v1/memories`): supply exactly one of `id` (single memory UUID) or `externalUserId` plus `format` (all memories for that user and format). Supplying both or neither returns 400; an unknown id returns 404.

## Safety

- Deleting memories is irreversible. Every delete requires explicit user approval; state exactly which form is used (`id` with the UUID, or `externalUserId` plus `format`) and what will be removed before calling.
- The migration prompt can contain sensitive user data (memory contents, scheduled-message reasons and context). Never log it, never paste it into unrelated channels, and only forward it to the destination the user named.
- Validate the delete form before calling: exactly one of `id` or `externalUserId` plus `format`. When deleting by user, `format` is required and must be `Text` or `DateReminder`.

## Validation

- List or get before deleting to confirm the target exists and belongs to the expected external user.
- After delete by id, confirm the id returns 404. After delete by user and format, confirm a fresh list is empty for that pair.
- After migration generation, confirm the prompt covers the expected external user before handing it off.
- The trace ID is preserved.

## Limits

- `format` accepts only `Text` or `DateReminder`; anything else returns 400.
- Delete accepts exactly one addressing form. Combining `id` with `externalUserId`, or omitting `format` on a user delete, returns 400.
- List previews truncate content to 120 characters; use the detail endpoint for full content.

## Escalation

- The need is conversation memory rather than persisted memory: load `references/ai-gateways/` and use gateway memory options instead.
- The format value is rejected: confirm it is exactly `Text` or `DateReminder` and retry.
- The delete target is ambiguous (wrong user, unknown id): stop, re-list with a `filter`, and ask the user to confirm the exact target.
- The migration prompt must go somewhere new: ask the user for the destination and do not store or forward it elsewhere.

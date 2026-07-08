# Safe Mutations

Use this file before changing AIVAX account state.

## Required Sequence

1. Read current state.
2. Identify the exact target resource ID.
3. Decide whether the change is reversible, destructive, cost-sensitive, or customer-facing.
4. Export or copy current configuration when rollback matters.
5. Patch only the fields that should change.
6. Verify resource state after the mutation.
7. Validate through the relevant product path.
8. Report what changed, what was verified, and what risk remains.

## Confirmation Required

Ask for explicit user confirmation before:

- Deleting resources.
- Resetting collections.
- Clearing all skills.
- Importing skills or documents in a mode that overwrites or deletes existing records.
- Rotating salts, keys, or secrets.
- Pausing, finishing, or cancelling active production jobs.
- Changing billing, plan, rate limits, storage cleanup, or cost controls.
- Changing production gateway model, routing, tool access, moderation, or chat delivery behavior when impact is uncertain.

## Mutation Rules

- Prefer `PATCH` with minimal fields when available.
- For shallow-merge endpoints, do not send copied full objects unless intentionally replacing the object.
- Treat arrays as replacements unless documentation proves merge semantics.
- Preserve external provider `apiKey` values when patching unrelated gateway fields.
- Preserve existing gateway skills unless intentionally removing them.
- Do not use local repository files or database internals to determine account state.

## Validation By Resource

- Gateway: view gateway, run test prompt, inspect resulting conversation.
- RAG: run `/api/v1/query`, inspect top results and transaction.
- Chat client: view client, create/refresh session when appropriate, verify talk URL without exposing access key.
- Batch: view workflow/job/items, inspect sample output, export only when needed.
- Skill: view skill, view gateway attachment, run test conversation, verify tool visibility.
- Cost: re-check usage after the relevant window; do not promise immediate cost deltas when usage data can lag.


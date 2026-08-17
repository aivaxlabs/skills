# Safe Mutations

Every AIVAX account mutation follows the same contract. The contract exists because most AIVAX endpoints shallow-merge, several accept destructive bulk operations, and many have user-facing blast radius. The contract is independent of which sub-skill triggers the mutation.

## Required Sequence

1. **Read current state.** Always `GET` the resource first. Confirm the ID is correct, the resource exists, and the current payload matches what the agent thinks it should be.
2. **Identify the exact target resource ID.** Never mutate by name; resolve to ID through the relevant list endpoint.
3. **Classify the change.** Reversible, destructive, cost-sensitive, or customer-facing. A change can be more than one.
4. **Export or copy current configuration** when rollback matters: gateway rewrites, skill imports, collection resets, batch workflow edits that alter validation, account or salt changes, production model swaps.
5. **Patch only the fields that should change.** For shallow-merge endpoints, do not send a copied full object unless the user asked for a full replacement and you verified the shape.
6. **Verify resource state after the mutation.** `GET` again and confirm the change landed only where intended.
7. **Validate through the relevant product path.** Gateway: a real or test conversation. RAG: `/api/v1/query` or a transaction. Chat client: a session or talk URL. Batch: items view or export. Skill: gateway attachment and a test conversation. Cost: re-check usage after the next window.
8. **Report what changed, what was verified, and what risk remains.** Include resource IDs, request IDs, and the next observation window.

## Approval Gates

Ask for explicit user confirmation before any of the following. The classification in the previous step is the source of truth; if in doubt, ask.

- Deleting resources (gateway, collection, document, skill, conversation, API key, chat client, session).
- Resetting collections or clearing all skills.
- Importing skills or documents in a mode that overwrites or deletes existing records (`insert-mode=sync`, broad skill imports).
- Rotating salts, hook keys, or other secrets.
- Pausing, finishing, or cancelling active production jobs.
- Changing plan, rate limits, storage cleanup, or cost controls.
- Changing production gateway model, routing, tool access, moderation, or chat delivery behavior when impact is uncertain.
- Bulk removing batch items or any operation that touches more than 100 records.

When a class of mutation is already explicitly authorized, do not re-ask. When the authorization is implicit (e.g. the user said "delete the broken gateway" and the broken gateway is the only target), apply it and report.

## Mutation Rules

- Prefer `PATCH` with minimal fields when available.
- For shallow-merge endpoints, do not send a copied full object unless intentionally replacing.
- Treat arrays as replacements unless documentation proves merge semantics. Gateway `parameters.skills` is a flat replacement; `tools` is usually a replacement; `knowledgeCollections` may be a replacement too. Verify with `aivax_search_context` when the array merge behavior is unclear.
- Preserve external provider `apiKey` values when patching unrelated gateway parameters.
- Preserve existing gateway skills unless intentionally removing them.
- Preserve `systemInstruction` when patching unrelated fields.
- Do not use local repository files or database internals to determine account state.

## Validation By Resource

- **Gateway**: view gateway, run a test prompt, inspect the resulting conversation.
- **RAG**: run `/api/v1/query`, inspect top results and a recent transaction.
- **Chat client**: view client, create or refresh a session when appropriate, verify talk URL without exposing access key.
- **Batch**: view workflow, job, items, inspect sample output, export only when needed.
- **Skill**: view skill, view gateway attachment, run a test conversation, verify tool visibility.
- **Cost**: re-check usage after the next relevant window; do not promise immediate cost deltas when usage data can lag.
- **API key**: list keys, confirm the new key has the intended type, label, and expiration; confirm the old key still works until rotation, then delete it only after the new one is validated end-to-end.

## Rollback Pattern

For most resources, the rollback is a `PUT` or `PATCH` that restores the captured pre-mutation payload. AIVAX does not have a generic "undo" endpoint. Capture, then restore — never re-derive the previous state from memory.

For skills, the rollback may be a re-import of the exported JSONL. For batch jobs, the rollback may be pausing the job and removing problematic items. For gateways, the rollback may be restoring `systemInstruction` and `parameters` from a previous view.

# Idempotent Retry

Use when a call mutates AIVAX state and may need to be retried. The patterns below prevent duplicates and silent overwrites.

## Objective

Retry a mutation safely. The retry must produce the same observable effect as a single successful call, even if the first call partially applied.

## Preconditions

- A mutation that needs to be retried (network error, transient 5xx, 408, 504).
- The mutation's pre-state is captured.

## Decision Tree

1. Does the endpoint accept an `idempotency_key`? If yes, use it to correlate the retry. Chat completions accept the AIVAX-specific `idempotency_key` (it sets the conversation's `ConversationToken` but does not deduplicate execution or cost); many other mutations do not accept any key.
2. Is the endpoint a `PUT` that overwrites the resource? If yes, a retry is safe only if the pre-state and the retry payload are identical. A retry that lands after a different mutation will overwrite the different mutation.
3. Is the endpoint a `POST` that creates a new resource? A retry may create a duplicate. Capture the new resource's ID and check for it before retrying (`idempotency_key` only applies to chat completions, where it correlates but does not suppress re-execution).
4. Is the endpoint a `DELETE`? A retry is safe (deleting an already-deleted resource is a no-op for most AIVAX resources, but verify with the affected sub-skill).
5. Is the endpoint a `PATCH` with shallow-merge? A retry is safe if the patch payload is identical and the resource state has not changed for unrelated reasons. If another mutation may have touched the same fields, do not retry — re-read the resource and re-decide.

## Actions

1. Capture the pre-mutation state of the resource (full `GET` response).
2. If the endpoint is chat completions, pass a stable AIVAX-specific `idempotency_key` to correlate the retry with the same conversation record (it does not prevent re-execution or charge).
3. Apply the retry. If the call fails again, inspect the resource to determine the actual state.
4. If the retry created a duplicate, plan a deduplication step (delete the duplicate, merge the records, or report the duplicate IDs to the user).
5. If the retry was a no-op, the original call was the one that landed. Re-validate.

## Validation

- The resource state matches the intended post-mutation state.
- No duplicate resources exist.
- The trace ID is preserved.
- The retry attempts are recorded in the trace.

## Limits

- The AIVAX-specific `idempotency_key` (chat completions only) is a string up to 128 characters that sets the conversation's `ConversationToken`. Make it stable per logical operation, not per physical call; it correlates but never deduplicates.
- A retry that lands after a different mutation (a colleague, a webhook, a user edit) is not idempotent. Re-read the resource.
- Some bulk operations (skill imports, document imports, batch item removals) cannot be retried safely. The rollback is the only recovery. See `references/platform-rules/safe-mutations.md`.

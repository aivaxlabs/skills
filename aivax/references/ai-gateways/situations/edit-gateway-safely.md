# Edit Gateway Safely

Use when the agent must change an AI Gateway's parameters with the smallest safe change.

## Objective

Change one thing at a time, preserve the rest, and validate the result through the same path the user will use.

## Preconditions

- The gateway exists and its current configuration is captured.
- The change is approved (when it affects production).
- The agent knows which field must change and which must be preserved.

## Decision Tree

1. Which field must change? Identify the exact field. The change should be a single field when possible.
2. Is the field at the top level of `parameters` or nested? Top-level fields are shallow-merged; nested objects may or may not be.
3. Is the field an array? Arrays are usually replaced during patching. Preserve existing IDs unless intentionally removing them.
4. Is the field a secret? `apiKey` for external providers is a secret. Preserve it when patching unrelated fields. Confirm with the user when changing it.
5. Will the change affect customer-facing behavior? If yes, confirm with the user.
6. Will the change affect cost? If yes, estimate the delta and confirm with the user.

## Construction

```text
GET /api/v1/ai-gateways/<gateway-id>      # capture the current configuration
PATCH /api/v1/ai-gateways/<gateway-id>   # patch only the field that should change
{
  "parameters": {
    "<field>": "<new-value>"
  }
}
GET /api/v1/ai-gateways/<gateway-id>      # confirm the change landed only where intended
```

For arrays, send the full intended array. Do not send a "diff" — the API does not apply per-element diffs.

For provider keys, do not echo the key in the request unless intentionally changing it.

## Validation

- The gateway view reflects the new value.
- Related fields are unchanged.
- A test conversation or inference smoke test produces the expected result.
- The trace ID is preserved.
- Recent conversations do not show new errors or unexpected token growth.

## Failure Modes

- The field is rejected: the value is the wrong type, the enum is wrong, or the field is read-only. Inspect the error and the field's expected shape with `aivax_search_context`.
- An array was unintentionally replaced: the patch sent a partial array. Re-patch with the full intended array.
- A secret was leaked in a request log: rotate the secret. Load `references/account/situations/create-and-rotate-keys.md`.
- The change broke a related feature: roll back. Load `references/platform-rules/safe-mutations.md`.

## Limits

- `PATCH` is shallow-merge on top-level `parameters` fields. Nested objects may or may not be merged. Verify with `aivax_search_context`.
- The agent should not assume recent conversations used the current config. The gateway may have been changed after the conversation ran.

## Escalation

- The change is a model swap: load `references/text-inference/situations/choose-model.md`.
- The change is a RAG tuning: load `references/rag/situations/design-rag-pipeline.md`.
- The change is a skill attachment: load `situations/attach-skills-and-tools.md`.
- The change broke a related feature: load `references/platform-rules/safe-mutations.md` and roll back.

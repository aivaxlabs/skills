# Error Handling

AIVAX errors fall into a small number of classes. The agent should classify before reacting, because the right response to a 429 is not the right response to a 401, and the right response to a 503 is not the right response to a 400. Detailed retry and resilience strategies live in `references/resilience/`; this file is the classification step.

## First Response Pattern

1. Preserve the exact status code, error code, and short message. Do not paraphrase away identifiers.
2. Do not retry mutations blindly. Re-read the target resource; the error may be caused by stale state.
3. Search AIVAX context (`aivax_search_context` or the public docs) if the error names an unknown field, endpoint, state, or enum.
4. If the error involves permission, auth, or plan limits, inspect account state before changing any resource.
5. Decide the class using the table below. Escalate to `references/resilience/` for the retry policy.

## Error Classes

### Authentication or Authorization (401, 403)

- Verify the MCP is configured for the intended account.
- Do not ask the user to paste secrets unless no safer account-auth surface exists.
- If using fallback HTTP, confirm the base URL, key type (`sk-aiv-acc` private vs `pk-aiv-` public), and the route is public-safe.
- Public keys are restricted to RAG query, answer, speech, image generation, media descriptions, and chat completions with stripped tool surfaces. They cannot call integrated models directly and cannot use gateway slugs.

### Not Found (404)

- Re-list resources and confirm the ID.
- Check whether the resource belongs to another account, environment, or region.
- Avoid creating duplicates until the missing resource is confirmed.

### Validation or Schema Error (400, 422)

- Compare the payload to the live API reference with `aivax_search_context`.
- Remove fields that were copied from a full object but are not part of the update.
- Check enum casing, array replacement semantics, and date format (ISO 8601 UTC).
- Re-read the current resource; the agent may have patched a field that has since been replaced or removed.

### Insufficient Balance, Quota, or Plan Limit (402, 429)

- Inspect `GET /api/v1/information/balance` and the relevant `usage` period.
- Quotas may be per-minute request, token-rate, or storage. Plans have commission multipliers (Free 1.25x, Pro 1.05x, Max 1.00x) that affect how balance translates to capacity.
- Do not start large jobs, performance reports, or media-heavy inference until cost is understood.
- For 429 specifically, load `references/resilience/situations/rate-limit-429.md`.

### Provider or Model Error (5xx with provider in the message)

- Use `aivax_list_models` to verify the model is available on the current plan.
- Inspect gateway model, base address, and api key without exposing secrets.
- Check recent conversations for repeated failures; if the failure is systemic, do not keep retrying.

### Timeout or Latency (504, 408, or slow 200)

- Inspect recent conversations, usage, RAG transactions, tool calls, and batch job state.
- Do not raise limits before finding the bottleneck; raise limits last.
- Cross-reference with `references/observability/situations/diagnose-degradation.md`.

### Tool or MCP Failure (tool or MCP surfaced inside a chat completion)

- Inspect the gateway's `mcpSources`, `protocolFunctions`, and `builtinFunctionsOptions`.
- Confirm the gateway has access to the tool (not hidden by `hideToolsWithoutSkill`).
- Inspect the conversation's tool calls and tool results; the model may be calling the wrong tool or calling it with the wrong arguments.

### Webhook or Hook Nonce Failure (worker, protocol function callback)

- Confirm the account has a hook key and that the service validates the BCrypt nonce.
- Rolling the salt invalidates existing worker and integration validation secrets.

## Error Recovery Discipline

- Never swallow an error class into a generic "failed" message. Report the class, the IDs, and the next action.
- Never log secrets when reporting an error. The hook key, provider key, and access key are always redacted.
- For a chain of calls, the first error stops the chain. Do not continue executing later steps if an earlier step failed; partial state may be more dangerous than full rollback.
- For destructive operations, the error response is the only signal that the operation may have been partially applied. Inspect the resource before deciding to retry.

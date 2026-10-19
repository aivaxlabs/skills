# Error Handling

AIVAX errors fall into a small number of classes. The agent should classify before reacting, because the right response to a 429 is not the right response to a 401, and the right response to a 503 is not the right response to a 400. Detailed retry and resilience strategies live in `references/resilience/`; this file is the classification step.

## First Response Pattern

1. Preserve the exact status code, available error code, safe short message, and `Retry-After`. Read either the standard `error` string with optional `details` or the OpenAI-compatible `error.message`, `error.type`, and `error.code` object. Auth/balance responses can use the standard envelope even on OpenAI-compatible routes.
2. Do not retry mutations blindly. Re-read the target resource; the error may be caused by stale state.
3. Search AIVAX context (`aivax_search_context` or the public docs) if the error names an unknown field, endpoint, state, or enum.
4. If the error involves permission, auth, or plan limits, inspect account state before changing any resource.
5. Decide the class using the guidance below. Escalate to `references/resilience/` for the retry policy.

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

### Invalid Request (400; endpoint-specific 422)

- Malformed input and request-content rejections from a model or dependency return `400`; do not retry the unchanged payload, even if the message mentions a temporarily unavailable service.
- Failures raised while the server reads or handles the HTTP request itself (unreadable request, dropped or cancelled client connection) are classified as client errors (`400`), not `500`. Fix the request or connection; do not retry it unchanged.
- A model with no eligible provider for the account's zero data retention restriction returns a client error naming that restriction, before inference and without charge. Choose another model or ask the user to review the account setting; do not retry.
- Treat `422` according to the endpoint contract; it is not the default schema-validation status.
- Compare the payload to the live API reference with `aivax_search_context`.
- Remove fields that were copied from a full object but are not part of the update.
- Check enum casing, array replacement semantics, and date format (ISO 8601 UTC).
- Re-read the current resource; the agent may have patched a field that has since been replaced or removed.

### Balance Or Storage Gate (402), Rate Limit (429)

- Inspect `GET /api/v1/information/balance` and the relevant `usage` period. `402` can indicate insufficient balance or exceeded storage; use the message to distinguish them.
- `429` indicates a rate limit, not a request to top up. Balance and usage are not live rate-limit counters, and `Retry-After` is not guaranteed on a rate-limit response.
- Consult [Pricing](https://docs.aivax.net/docs/pricing) and [Plans and limits](https://docs.aivax.net/docs/limits), not fixed commercial figures in a skill. Do not start large jobs until cost is understood.
- For `429`, load `references/resilience/situations/rate-limit-429.md`.

### Unavailable Dependency (503)

- Dependency failures and HTTP dependency timeouts normally return `503` with `Retry-After`. A vendor name is not required in the message; classify by status and code, not provider-name matching.
- Dependency-side auth, balance, timeout, and throttling failures can surface as `503`; do not mistake them for the caller's `401`, `402`, `403`, or `429`.
- For model failures, verify availability with `aivax_list_models` and inspect gateway model, base address, and key configuration without exposing secrets.
- Use bounded retries through `references/resilience/situations/transient-failure.md`; honor `Retry-After` and inspect mutation state first.

### Unexpected AIVAX Failure (500)

- The response uses a generic message, not internal failure details. Request-reading and client-disconnect problems are reported as client errors, not `500`; do not treat a `500` as a payload-validation failure or guess a root cause from the text.
- Preserve request identifiers; retry only when safe and bounded. Escalate repeated failures with `references/observability/situations/diagnose-degradation.md`.

### Timeout Or Slow Response

- Do not require `408` or `504` to identify a dependency timeout; it normally surfaces as `503`. If another layer returns a timeout status or the connection drops, preserve that evidence and inspect operation state before retrying.
- Inspect recent conversations, usage, RAG transactions, tool calls, and batch job state. Do not raise limits before finding the bottleneck.
- Cross-reference with `references/observability/situations/diagnose-degradation.md`.

### OpenAI-Compatible Error Codes And Streams

- For HTTP exception responses, `error.code` is `invalid_request_error` for client failures, `service_unavailable` for dependency failures, or `server_error` for unexpected AIVAX failures. Use HTTP status to distinguish auth, balance, permission, and rate-limit cases within `invalid_request_error`.
- Once chat-completion streaming starts, inspect error chunks even after a successful HTTP status. A chunk can contain `finish_reason: "error"` and an `error` object before `[DONE]`; its code may be provider-specific. Neither the initial status nor `[DONE]` proves successful generation.

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
- An error response or lost connection does not prove that a mutation had no effect. Inspect the resource before deciding to retry.

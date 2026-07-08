# MCP And API Errors

Use this file when an AIVAX MCP or account API call fails.

## First Response Pattern

1. Preserve the exact status, error code, and short message.
2. Do not retry mutations blindly.
3. Search AIVAX context if the error names an unknown field, endpoint, state, or enum.
4. Re-read the target resource if the error may be caused by stale state.
5. If the error involves permission, auth, or plan limits, inspect account/balance/model availability before changing resources.

## Common Classes

Authentication or authorization:

- Verify the MCP is configured for the intended account.
- Do not ask the user to paste secrets unless no safer account-auth surface exists.
- If using fallback HTTP, confirm base URL and auth mechanism explicitly.

Not found:

- Re-list resources and confirm ID.
- Check whether the resource belongs to another account/environment.
- Avoid creating duplicates until the missing resource is confirmed.

Validation or schema error:

- Compare payload to current API docs with `aivax_search_context`.
- Remove fields that were copied from a full object but are not part of the update.
- Check enum casing and array replacement semantics.

Insufficient balance or plan limit:

- Inspect `GET /api/v1/information/balance`.
- Inspect usage for the relevant period.
- Do not start large jobs or performance reports until cost is understood.

Provider or model error:

- Use `aivax_list_models` to verify model availability.
- Inspect gateway model/baseAddress/apiKey configuration without exposing secrets.
- Check recent conversations for repeated failures.

Timeout or latency:

- Inspect recent conversations, usage, RAG transactions, tool calls, and batch job state.
- Do not raise limits before finding the bottleneck.


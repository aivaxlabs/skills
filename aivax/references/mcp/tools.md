# MCP Tools

Use this file to choose and shape calls through the AIVAX Account Management MCP.

## Tools

### `aivax_invoke_function`

Use for account-scoped API operations.

Shape:

```text
aivax_invoke_function({
  "method": "GET",
  "path": "/api/v1/ai-gateways"
})
```

Rules:

- Use hostless paths only, such as `/api/v1/collections`.
- Include query strings in `path`.
- Put JSON request payloads in `body`.
- Do not include account secrets, provider keys, or authorization headers unless the MCP contract explicitly asks for them.
- Read current state before `POST`, `PUT`, `PATCH`, or `DELETE`.

### `aivax_list_models`

Use before selecting or changing models.

Shape:

```text
aivax_list_models({
  "name_filter": "gpt"
})
```

Check:

- Provider and model name.
- Plan availability.
- Pricing.
- Context window and output limits.
- Tool, multimodal, reasoning, structured-output, and speed characteristics.
- Preview, deprecated, or unstable flags.

### `aivax_search_context`

Use before acting on unfamiliar endpoints, payloads, enum values, or features.

Shape:

```text
aivax_search_context({
  "query": "AI gateway queryStrategy parameters",
  "search_type": "all"
})
```

Use `api-function-reference` for endpoint shapes, `documentation-manual` for product behavior, and `all` when unsure.

## Call Order Pattern

1. Search context if the API shape is uncertain.
2. List candidate resources.
3. View the exact target resource.
4. Apply the smallest mutation.
5. View the resource again.
6. Validate through a user-facing or telemetry path.


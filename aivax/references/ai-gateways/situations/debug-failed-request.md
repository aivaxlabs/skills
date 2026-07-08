# Debug Failed Gateway Request

Use when the user says an AIVAX gateway request failed, returned an error, produced no answer, called the wrong tool, or used the wrong model.

Goal: identify whether the failure is request shape, gateway configuration, model availability, provider error, tool/MCP failure, RAG context, moderation, balance, or chat/client state.

## Steps

1. Identify gateway ID, approximate time, channel, and expected behavior.
2. List recent conversations for the gateway:

```text
GET /api/v1/conversations?offsetminutes=1440&filter=--gateway <gateway-id>
```

3. View the failing conversation:

```text
GET /api/v1/conversations/<conversation-id>
```

4. Inspect `errorMessage`, `modelName`, `usageObject`, `tools`, `resources`, messages, and timestamps.
5. View the gateway:

```text
GET /api/v1/ai-gateways/<gateway-id>
```

6. If model availability is suspicious, call `aivax_list_models`.
7. If tool behavior is suspicious, inspect gateway `skills`, `hideToolsWithoutSkill`, `alwaysVisibleTools`, `mcpSources`, and `protocolFunctions`.
8. If RAG behavior is suspicious, load `references/rag/situations/inspect-retrieval.md`.
9. Apply the smallest fix only after confirming the resource causing the failure.

## Common Causes

- Model not available on the plan.
- External provider key invalid or omitted during patch.
- Tool hidden by skill/tool visibility rules.
- MCP source unreachable or wrong tool schema.
- RAG collection missing, stale, or too restrictive.
- Context overflow or bad truncation setting.
- Balance or subscription reserve exhausted.
- Moderation or structured-output validation blocked the request.

## Validation

- Run one minimal representative request through the same gateway.
- View the new conversation.
- Confirm the expected model, resources, tools, and no repeated error.
- If a customer-facing channel failed, test through that same channel or chat client.


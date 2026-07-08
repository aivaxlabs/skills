# Analyze Gateway Latency

Use when the user says a gateway, chat client, or model response is slow.

Goal: identify whether latency comes from model choice, RAG, tools, context size, output length, provider behavior, chat channel behavior, or batch load.

## Steps

1. Get recent conversations for the gateway, client, or session.
2. View representative slow conversations.
3. Inspect timestamps, token counts, model name, tool calls, resources, and RAG transaction references.
4. Inspect gateway parameters:
   - model
   - max output tokens
   - context size and overflow action
   - RAG top K and references
   - tool context count
   - MCP/tool sources
5. If RAG is involved, inspect collection transactions for process time.
6. If tools are involved, compare tool count and repeated calls.
7. If channel-specific, inspect chat client session and integration settings.
8. Recommend the smallest change.

## Common Causes

- Expensive or slow model.
- Excessive retrieved documents or referenced parent documents.
- Repeated tool calls or large tool results.
- Long session context or large `extraContext`.
- Long output limit or verbose system instructions.
- External provider latency.
- Batch or subscription reserve pressure.

## Validation

- Run the same short prompt before and after change when safe.
- Compare conversation timing, tokens, tool count, and model name.
- Do not claim improvement from a single sample if latency is variable.


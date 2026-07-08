# Inspect Gateway Routing

Use when the user wants to understand which model, gateway, skills, RAG collections, tools, or routing behavior is active.

Goal: report the actual configured route and the route observed in recent conversations.

## Steps

1. List gateways if the target is not known:

```text
GET /api/v1/ai-gateways
```

2. View the gateway:

```text
GET /api/v1/ai-gateways/<gateway-id>
```

3. Inspect:
   - `parameters.baseAddress`
   - `parameters.modelName`
   - `parameters.systemInstruction`
   - `parameters.skills`
   - `parameters.knowledgeCollections`
   - `parameters.mcpSources`
   - `parameters.protocolFunctions`
   - `parameters.contextMaximumSize`
   - `parameters.contextOverflowAction`
4. List recent conversations for the gateway.
5. Confirm the actual `modelName` used in conversations.
6. Map skills and collections by viewing `/api/v1/skills` and `/api/v1/collections` when IDs are not self-explanatory.

## Gotchas

- Gateway parameter arrays are usually replacements during patching.
- Account skills are attached by ID, not slug.
- `systemInstruction` is not the same thing as account skills.
- Integrated models should use `baseAddress: "@integrated"` with a model returned by `aivax_list_models`.
- Do not assume recent conversations used the current config if the gateway was changed after they ran.

## Validation

Report both configured route and observed route. If they differ, include timestamp and likely reason.


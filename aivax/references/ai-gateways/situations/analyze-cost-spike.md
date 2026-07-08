# Analyze Gateway Cost Spike

Use when a gateway or related agent suddenly costs more than expected.

Goal: connect spend to model, gateway settings, RAG, tools, chat sessions, or batch usage.

## Steps

1. Get usage for 24h, 7d, or the user-provided window:

```text
GET /api/v1/information/usage?timeStart=<iso-start>&timeEnd=<iso-end>
```

2. Identify top model, category, and resource drivers.
3. Map resource IDs back to gateway, chat client, collection, batch workflow/job, or API key.
4. View the gateway and recent conversations.
5. Inspect token counts, output length, RAG top K, tool calls, model name, and chat sessions.
6. Load `references/cost-monitoring/situations/investigate-cost-spike.md` for broader spend analysis.

## Common Causes

- Model changed to a more expensive model.
- Long outputs or high `maxCompletionTokens`.
- RAG injected too many documents.
- Tool loop or repeated tool calls.
- Batch job attached to a costly gateway.
- Chat session context grew too large.
- Subscription reserve exhausted and fallback pricing applies.

## Validation

- Confirm the resource attribution in usage data.
- Inspect at least one representative conversation or job item.
- Recommend changes ordered by savings and behavior risk.
- Get approval before changing model, limits, RAG volume, tools, or active jobs.


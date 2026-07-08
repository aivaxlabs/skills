# Cost Monitoring

Use this sub-skill to explain where AIVAX spend and consumption come from, then recommend safe optimization actions. Work from account usage data, balance, model pricing, resource attribution, conversations, RAG transactions, and batch job costs.

This is for monitoring and optimizing account consumption through AIVAX usage surfaces. Do not inspect billing source code, database internals, or platform implementation unless the user explicitly asks for source-code work.

## Situation Playbooks

- `situations/investigate-cost-spike.md`: balance drop, reserve exhaustion, runaway resource spend, or unexpected model/batch/RAG cost.

Use the listed API paths through `aivax_invoke_function` with hostless paths. Search AIVAX context before using unfamiliar usage fields, resource filters, plan limits, or billing categories.

```text
aivax_invoke_function({
  "method": "GET",
  "path": "/api/v1/information/balance"
})
```

## Current Account Health

Start with balance and quota status:

```text
GET /api/v1/information/balance
```

Review:

- `balance`: current prepaid balance.
- `usage24h`: recent spend.
- `plan`: account plan.
- `storageUsage`: current account storage usage.
- `subscriptionModelUsage`: six-hour and weekly reserve consumption for subscription models.
- `planLimits`: storage and subscription-model availability.

## Usage Period Analysis

Use usage details for a period. Prefer relative windows such as last 24 hours, last 7 days, and last 30 days, converted to ISO timestamps in the account/user timezone when known:

```text
GET /api/v1/information/usage?timeStart=<iso-start>&timeEnd=<iso-end>
```

The response includes:

- `currentBalance` and invoices.
- `usageDetails.modelUsageDiscriminations`: spend and token counts by model/category.
- `usageDetails.categoryDiscriminations`: spend by SKU/category.
- `usageDetails.resourceDiscriminations`: spend by resource type and resource ID/name.
- `usageDetails.meterGroups`: storage, period spend, token volume, input tokens, output tokens, cached tokens, and cache hit rate.
- `usageDetails.charts`: time-series usage by model, category, and resource unless the response is truncated.
- `usageDetails.dayListings`: recent usage rows grouped by day.

Use ISO timestamps and keep the period under one year. URL-encode filter values that include spaces or flags, such as `--gateway <gateway-id>`. For a quick audit, compare 24h, 7d, and 30d windows.

## Model Pricing and Availability

Use model metadata before recommending model swaps:

```text
aivax_list_models({ "name_filter": "sonnet" })
```

Compare input, cached input, output, audio input prices, context window, max output tokens, speed, intelligence, stability, preview/deprecated flags, plan availability, and capabilities.

Never recommend a cheaper model if it lacks required capability, context window, modality, or tool behavior.

## Spend Driver Workflow

1. Get account balance and usage for the relevant period.
2. Rank drivers by model, category, and resource.
3. Map resource IDs back to gateways, collections, chat clients, sessions, batch workflows, or API keys.
4. Inspect high-spend resources:

```text
GET /api/v1/ai-gateways/<gateway-id>
GET /api/v1/web-chat-client/<client-id>
GET /api/v1/web-chat-client/<client-id>/sessions?filter=<session-id>
GET /api/v1/collections/<collection-id>/transactions?view=recent
GET /api/v1/batch/workflows/<workflow-id>/jobs/<job-id>
GET /api/v1/conversations?offsetminutes=10080&filter=--gateway <gateway-id>
```

5. Identify the controllable cause.
6. Propose the smallest change; apply it only when the user already authorized that class of mutation.
7. Re-check usage after the next relevant window when possible.

Require explicit approval before model swaps, batch pause/finish, rate-limit changes, debug disabling, upload-mode changes, storage cleanup, collection deletion, skill/tool restrictions, or any change that may reduce answer quality, stop active work, delete data, or alter a customer-facing chat experience.

## Optimization Levers

Model and gateway:

- Replace overpowered models with cheaper compatible models.
- Use model routing when request complexity varies.
- Lower `maxCompletionTokens` when outputs are longer than needed.
- Use `contextMaximumSize` and an appropriate `contextOverflowAction`.
- Keep `reasoningEffort` proportional to task complexity.

RAG:

- Lower `knowledgeBaseMaximumResults` when extra documents do not improve answers.
- Raise `knowledgeBaseMinimumScore` when irrelevant documents inflate context.
- Disable `knowledgeUseReferences` unless referenced parents materially improve grounding.
- Tune `queryStrategy` instead of blindly increasing retrieval volume.
- Use collection transactions and quality reports before reindexing or rewriting large corpora.

Tools:

- Enable only necessary built-in functions, MCP tools, and protocol functions.
- Use `hideToolsWithoutSkill` with carefully designed skills for tool-heavy gateways.
- Investigate repeated tool calls in conversations before raising tool context or model limits.
- Limit `toolContextCount` when old tool results inflate prompts.

Chat clients and sessions:

- Set sensible `messagesPerHour` and `maxMessages`.
- Avoid large repeated `extraContext`; use concise session metadata.
- Disable debug modes and unnecessary upload modes after troubleshooting.
- Review integration debounce and scheduled-continuation settings if channel usage spikes.

Batch:

- Run a small sample before activating large jobs.
- Use strict result schemas and validation only when the business value justifies the extra calls.
- Export low-confidence and error items before retrying.
- Retry targeted modes rather than all failed items.

Storage:

- Use collection details and document browsing to find stale collections or oversized corpora.
- Prefer targeted document cleanup over collection reset.
- Delete collections only with explicit user approval and export if retention matters.

## Runaway Usage Checks

Investigate immediately when 24h usage spikes, one resource dominates spend, a batch job has many retries, conversations show repeated tool calls, RAG transactions are slow and low-quality, or subscription reserve is exhausted early.

Pause or finish batch jobs before changing workflows if they are actively driving cost.

## Reporting

Return period analyzed, balance and 24h spend, top spend drivers, likely causes, recommended actions ordered by savings and risk, changes applied, and validation or follow-up measurement plan.

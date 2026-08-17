# Inspect Balance And Usage

Use when the agent must read account state — balance, plan, storage, reserve consumption, or detailed usage for a period — to inform a decision.

## Objective

Know the account's current capacity and recent consumption before any operation that costs balance or quota.

## Preconditions

- The agent has access to the account (MCP or direct HTTPS with a private key).
- The period of interest is defined (e.g. last 24 hours, last 7 days, last 30 days, or a custom range).

## Decision Tree

1. Is the question about the current health (balance, plan, storage, reserve)? Call `GET /api/v1/information/balance` and read the top-level fields.
2. Is the question about a period (model spend, category spend, resource spend, charts)? Call `GET /api/v1/information/usage?timeStart=<iso-start>&timeEnd=<iso-end>` and read the detailed fields.
3. Is the period longer than one year? Split it into smaller windows. The endpoint may truncate.
4. Are the resource IDs in the response meaningful? Map them back to gateways, collections, chat clients, sessions, batch workflows, or API keys. The agent has those resources; map them.
5. Is the storage usage approaching the plan limit? Plan limits control storage. Confirm with the user before deleting or resetting.

## Construction

```text
GET /api/v1/information/balance
GET /api/v1/information/usage?timeStart=<iso-start>&timeEnd=<iso-end>
```

For multiple windows, call the usage endpoint multiple times with different ranges and compare.

## Reading The Response

The balance response includes:

- `balance`: current prepaid balance.
- `usage24h`: recent spend.
- `plan`: account plan.
- `storageUsage`: current account storage usage.
- `subscriptionModelUsage`: six-hour and weekly reserve consumption for subscription models.
- `planLimits`: storage and subscription-model availability.

The usage response includes:

- `currentBalance` and invoices.
- `usageDetails.modelUsageDiscriminations`: spend and token counts by model and category.
- `usageDetails.categoryDiscriminations`: spend by SKU and category.
- `usageDetails.resourceDiscriminations`: spend by resource type and resource ID.
- `usageDetails.meterGroups`: storage, period spend, token volume, input tokens, output tokens, cached tokens, and cache hit rate.
- `usageDetails.charts`: time-series usage by model, category, and resource unless the response is truncated.
- `usageDetails.dayListings`: recent usage rows grouped by day.

## Validation

- The response is parsed and the relevant fields are extracted.
- The period matches the agent's intent.
- The resource IDs are mapped back to the agent's known resources.
- The trace ID is preserved.

## Limits

- The period should be under one year. Larger periods may truncate.
- URL-encode filter values that include spaces or flags (e.g. `--gateway <gateway-id>`).
- Usage data can lag real-time spend by a few minutes. Do not promise immediate cost deltas.

## Escalation

- Balance is low: load `references/cost-monitoring/situations/optimize-spend.md`.
- A spike is detected: load `references/cost-monitoring/situations/investigate-cost-spike.md`.
- Storage is approaching the plan limit: identify the largest collections and consider cleanup. Confirm with the user.
- Subscription reserve is exhausted early: identify the dominant consumer (a model, a gateway, a session) and reduce its consumption. Confirm with the user.

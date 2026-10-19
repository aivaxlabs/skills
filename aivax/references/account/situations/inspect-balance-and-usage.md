# Inspect Balance And Usage

Use when the agent must read account state — balance, plan, storage, subscription allowance consumption, or detailed usage for a period — to inform a decision.

## Objective

Know the account's current capacity and recent consumption before any operation that costs balance or quota.

## Preconditions

- The agent has access to the account (MCP or direct HTTPS with a private key).
- The period of interest is defined for the requested comparison.

## Decision Tree

1. Is the question about the current health (balance, plan, storage, allowances)? Call `GET /api/v1/information/balance` and read the top-level fields.
2. Is the question about a period (model spend, category spend, resource spend, charts)? Call `GET /api/v1/information/usage?timeStart=<iso-start>&timeEnd=<iso-end>` and read the detailed fields.
3. Is the period beyond the endpoint's accepted window? Split it into smaller windows; oversized periods are rejected.
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
- `subscriptionModelUsage`: service-allowance snapshot with `consumptions` by type, `usedPercent`, `resetAt`, `isExhausted`, and usage-share `details`. Inference coverage is currently disabled.
- `planLimits`: included storage and whether the plan defines a subscription-model allowance; this does not mean inference coverage is active.

The usage response includes:

- `currentBalance` and `invoices`: each has `invoiceId`, `createdAt`, `paidAt`, `expiresAt`, `amount`, `description`, `isExpired`, `type` (`CreditAddition`, `SubscriptionPayment`, or `SubscriptionRenewal`), `paymentLocation` (only while the invoice is open), and `receiptLocation` (only when paid). Only `CreditAddition` invoices add prepaid balance; do not read subscription invoices as top-ups. Do not echo payment or receipt links in reports; cite the invoice ID and state. Load `situations/manage-subscription.md` to act on them.
- `usageDetails.modelUsageDiscriminations`: spend and token counts by model and category.
- `usageDetails.categoryDiscriminations`: spend by SKU and category.
- `usageDetails.resourceDiscriminations`: spend by resource type and resource ID.
- `usageDetails.meterGroups`: storage, period spend, token volume, input tokens, output tokens, cached tokens, and cache hit rate.
- `usageDetails.charts`: time-series usage by model, category, and resource, omitted when the `X-Response-Truncating: agent-optimized` header is used.
- `usageDetails.dayListings`: recent usage rows grouped by day.

## Validation

- The response is parsed and the relevant fields are extracted.
- The period matches the agent's intent.
- The resource IDs are mapped back to the agent's known resources.
- The trace ID is preserved.

## Limits

- Oversized periods are rejected; split them into accepted windows.
- URL-encode filters when mapping resource IDs through conversation listings; the usage endpoint accepts time filters and `dayCount`.
- The balance view is cached briefly; usage reflects recorded billing rows, not allowance-covered items. Do not promise immediate cost deltas.

## Escalation

- Balance is low: load `references/cost-monitoring/situations/optimize-spend.md`.
- An invoice is open or a renewal failed: load `situations/manage-subscription.md`.
- A spike is detected: load `references/cost-monitoring/situations/investigate-cost-spike.md`.
- Storage is approaching the plan limit: identify the largest collections and consider cleanup. Confirm with the user.
- A service allowance is exhausted early: inspect its consumption details and reset time; items that do not fit are billed normally. Compare billed usage before recommending a change.

# Investigate Cost Spike

Use when the user asks why AIVAX spend increased, balance dropped, reserve was exhausted, or a resource became expensive.

Goal: identify the cost driver, connect it to account resources, and recommend safe savings actions.

## Steps

1. Get current account health:

```text
GET /api/v1/information/balance
```

2. Get usage for the relevant period:

```text
GET /api/v1/information/usage?timeStart=<iso-start>&timeEnd=<iso-end>
```

3. Compare 24h, 7d, and 30d when the user did not specify a window.
4. Rank spend by model, category, and resource.
5. Map top resource IDs to gateways, collections, chat clients, API keys, batch jobs, or conversations.
6. Inspect representative high-cost resources.
7. Identify controllable causes.
8. Recommend changes ordered by savings and behavior risk.

## Common Causes

- Expensive model or model swap.
- Long prompts, long outputs, or context overflow handling.
- Too many RAG results or parent references.
- Tool loops or large tool results.
- Batch retry storm.
- Chat session context growth.
- Storage growth from stale collections.
- Subscription reserve exhausted.

## Approval Gates

Get explicit approval before:

- Model swaps.
- Pausing or finishing active jobs.
- Lowering limits.
- Deleting storage or collections.
- Restricting tools or skills.
- Changing customer-facing chat behavior.

## Validation

- Confirm the cost driver in usage data.
- Confirm the resource settings that caused it.
- If a change is applied, re-check usage after the next relevant reporting window.


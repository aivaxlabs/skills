# Investigate Cost Spike

Use when the user asks why AIVAX spend increased, balance dropped, reserve was exhausted early, or a resource became unexpectedly expensive.

## Objective

Identify the cost driver, connect it to account resources, and recommend safe savings actions — without changing anything before the user approves.

## Preconditions

- The user can describe the symptom (balance drop, spike, reserve exhaustion, expensive resource).
- A time window is defined (last 24 hours, last 7 days, last 30 days, or a custom range).
- The agent has access to the account (MCP or direct HTTPS with a private key).

## Decision Tree

1. What is the current account health? Call `GET /api/v1/information/balance` and read the top-level fields.
2. What is the usage for the relevant period? Call `GET /api/v1/information/usage?timeStart=<iso-start>&timeEnd=<iso-end>` and read the detailed fields.
3. Compare 24h, 7d, and 30d when the user did not specify a window. The dominant driver often appears in a longer window.
4. Rank spend by model, category, and resource. The top driver is usually the cause.
5. Map top resource IDs to gateways, collections, chat clients, sessions, batch workflows, or API keys.
6. Inspect the top resources. For each: view the resource, then list recent conversations, transactions, or job items.
7. Identify the controllable cause. A controllable cause is one the agent can change safely.
8. Propose the smallest change. Order by savings and behavior risk.
9. Get explicit approval before any mutation. Do not act on a recommendation without approval.

## Common Causes

- Expensive model or model swap.
- Long prompts, long outputs, or context overflow handling.
- Too many RAG results or referenced parent documents.
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

- The cost driver is confirmed in usage data.
- The resource settings that caused the cost are confirmed in the resource view.
- The recommended change is ordered by savings and behavior risk.
- The user has approved the change before any mutation.
- After the change, the next usage window reflects the expected cost delta.

## Limits

- Usage data can lag real-time spend by a few minutes.
- A cost spike may be the intended effect of a previous change. Verify the history before reverting.
- Some optimizations are destructive. Confirm with the user.

## Escalation

- The cause is a runaway resource: load `references/observability/situations/diagnose-degradation.md` to find the loop.
- The cause is a model choice: load `references/text-inference/situations/choose-model.md` to compare alternatives.
- The cause is a RAG pipeline: load `references/rag/situations/design-rag-pipeline.md` to tune.
- The cause is a chat client or session: load `references/web-chat/` to inspect sessions.
- The cause is a batch retry storm: load `references/batch/situations/debug-failed-job.md`.

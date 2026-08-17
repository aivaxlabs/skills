---
name: aivax-cost-monitoring
description: Investigate AIVAX cost spikes, balance drops, runaway resource spend, and optimize account consumption through usage data, model pricing, resource attribution, conversations, RAG transactions, and batch job costs. Load when the user asks why AIVAX spend changed, or when the agent is deciding a model swap, a rate-limit change, or a resource cleanup.
---

# Cost Monitoring

This sub-skill owns cost and consumption on AIVAX. The discipline is to work from account usage data (not from guesses), connect the spend to specific resources, and recommend safe optimizations that the user has approved.

## Operating Files

- `situations/investigate-cost-spike.md`: balance drop, reserve exhaustion, runaway resource spend, or unexpected model/RAG/batch cost.
- `situations/optimize-spend.md`: identify safe levers to reduce spend without harming answer quality or user experience.

## When To Use Cost Monitoring

Use this sub-skill when the user asks:

- Why did AIVAX spend increase?
- Why did the balance drop?
- Why was the subscription reserve exhausted early?
- Why is one resource more expensive than the others?
- How can I reduce spend without changing user-visible behavior?

Do not use this sub-skill for:

- Inspecting the current balance in isolation (use `references/account/situations/inspect-balance-and-usage.md`).
- Selecting a model for a new task (use `references/text-inference/situations/choose-model.md`).
- Tuning RAG or batch for a specific result (use `references/rag/` or `references/batch/`).

## Operating Surfaces

Cost monitoring works through account usage data and resource attribution:

- `GET /api/v1/information/balance`: current health.
- `GET /api/v1/information/usage?timeStart=<iso>&timeEnd=<iso>`: detailed usage for a period.
- `GET /api/v1/ai-gateways/<id>`, `GET /api/v1/collections/<id>`, `GET /api/v1/web-chat-client/<id>`, `GET /api/v1/batch/workflows/<id>/jobs/<id>`, `GET /api/v1/conversations`: resource-level inspection.
- `aivax_list_models`: model pricing and capability.

## Plan Multipliers

Plans have commission multipliers that affect how balance translates to capacity:

- Free: 1.25x.
- Pro: 1.05x.
- Max: 1.00x.

The multiplier is applied to the per-unit cost; the public `usage.cost` is the final amount recorded for the account after applicable account adjustments.

## Cost Discipline

- Never recommend a model swap if the alternative lacks the required capability, context window, modality, or tool behavior.
- Never recommend a cheaper gateway for a production workload that is currently passing quality checks.
- Never delete a collection, reset a batch job, or pause an active job without explicit user approval.
- Never raise a rate limit or a quota without first finding the bottleneck.

## Validation

- The cost driver is confirmed in usage data.
- The resource settings that caused the cost are confirmed in the resource view.
- The recommended change is ordered by savings and behavior risk.
- The user has approved the change before any mutation.
- After the change, the next usage window reflects the expected cost delta.

## Limits

- Usage data can lag real-time spend by a few minutes. Do not promise immediate cost deltas.
- A change in cost may be the intended effect of a previous change (a model swap, a new gateway, a new chat client). Verify the history before reverting.
- Some optimizations (storage cleanup, batch finish, model swap) are destructive. Confirm with the user.

## Escalation

- A cost spike is unexplained: load `situations/investigate-cost-spike.md`.
- A cost optimization is requested: load `situations/optimize-spend.md`.
- A new resource is unusually expensive: load `references/observability/situations/diagnose-degradation.md` and confirm the resource is not in a failure loop.
- A subscription reserve is exhausted: identify the dominant consumer and reduce its consumption. Confirm with the user.

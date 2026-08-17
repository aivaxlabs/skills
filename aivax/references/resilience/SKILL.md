---
name: aivax-resilience
description: Reusable patterns for AIVAX API failures — rate limits (429), transient errors, partial failures, idempotent retries, fallback models, and graceful degradation. Load when a sub-skill encounters a retryable error or when the agent is designing a pipeline that must survive transient failures.
---

# Resilience

Any AIVAX sub-skill can fail. The patterns for surviving failure are reusable: they do not depend on which capability is calling. This sub-skill owns those patterns.

## Operating Files

- `situations/rate-limit-429.md`: how to handle 429 from any endpoint.
- `situations/transient-failure.md`: 5xx, 408, 504, and similar temporary errors.
- `situations/idempotent-retry.md`: how to retry without creating duplicates.
- `situations/partial-failure.md`: a chain of calls where one stage failed mid-way.

## Scope Boundary

This sub-skill defines the patterns. The decision of when to apply them belongs to the calling sub-skill. A pipeline that combines stages should load this sub-skill once and use the same pattern across all stages; do not invent a per-stage retry policy.

## Resilience Decision Order

1. **Is the error class transient?** Load `situations/transient-failure.md`. If not, do not retry — the error will not resolve on its own.
2. **Is the error class a rate limit?** Load `situations/rate-limit-429.md`. Apply exponential backoff with jitter; respect the `Retry-After` header when present.
3. **Is the call idempotent?** Load `situations/idempotent-retry.md`. If the call mutates state and is not idempotent, retry only with user approval.
4. **Did a chain fail partway through?** Load `situations/partial-failure.md`. Decide whether to rollback, skip the failed stage, or report partial state.

## Universal Rules

- **Exponential backoff with jitter.** Start at 500ms, double, cap at 30s, add up to 250ms of random jitter. Three attempts is usually enough; more than five is rarely worth it.
- **Honor Retry-After.** When AIVAX returns `Retry-After`, sleep for that duration before retrying. Do not retry earlier.
- **Idempotency for mutations.** Use `idempotency_key` on chat completions. For other mutations, capture the pre-mutation state and design a manual rollback in case the retry also fails.
- **Circuit breaker for known-bad states.** If three consecutive retries fail with the same class, stop retrying. Switch to a fallback (different model, different region) or surface the error to the user.
- **Cost-aware retries.** Do not retry large inference calls or batch operations on a 429 without confirming the user is willing to absorb the retry cost. Balance and quotas can be exhausted by a retry storm.
- **Time-boxed retries.** A retry that has not succeeded in 60 seconds is unlikely to succeed. Surface it.

## Fallback Strategies

When a primary call cannot be retried, the fallback options are:

- **Model swap**: a cheaper or faster model that meets the same capability floor. Verify the alternative model supports the modality, context, and tool surface.
- **Reranker swap**: when the reranker is failing, swap to `lexical` (no cost, no quota) or `none` (RAG only, disables rerank).
- **Gateway bypass**: a direct chat completion to an integrated model may bypass a gateway that is failing.
- **Skip stage**: a stage that is not critical (e.g. rerank, classify) can be skipped if the rest of the pipeline can still meet the goal.
- **Defer**: when the work is not time-critical, queue it for later rather than retrying now. Batch is the right shape for this.

## Cost Awareness

A retry is not free. Each attempt consumes balance, quota, and time. The decision to retry must include:

- Estimated cost of the retry.
- Probability the retry will succeed (drops sharply with each attempt).
- Cost of failing the user-facing task if the retry also fails.
- User signal: did the user accept "this may take longer"?

When the cost of the retry exceeds the cost of failing, surface the error and ask the user. Do not silently spend balance.

## Validation

- The retry succeeded (or the user accepted the failure).
- The trace ID was preserved across the retry.
- The cost report includes the retry attempts.
- The circuit-breaker or fallback was documented in the final report.

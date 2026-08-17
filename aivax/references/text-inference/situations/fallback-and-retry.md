# Fallback And Retry

Use when a text-inference call fails or is likely to fail, and the agent must decide whether to retry, swap, or surface the error.

## Objective

Recover from failure without losing the user's work, without spending more than the user accepted, and without making the failure worse.

## Preconditions

- A failing call (or a call that is at risk of failing based on prior evidence).
- Knowledge of the user's cost and latency caps.

## Decision Tree

1. Classify the error. Load `references/platform-rules/error-handling.md` if not yet classified.
2. Is the error class transient? Load `references/resilience/situations/transient-failure.md` and apply the retry policy.
3. Is the error a 429? Load `references/resilience/situations/rate-limit-429.md` and apply the rate-limit policy.
4. Is the error a model or provider error (model not available, model returned an error class)? Decide whether to swap to a fallback model.
5. Is the error a validation or schema error? The fix is not a retry. Adjust the payload, the schema, or the model.
6. After three failed retries, open the circuit breaker. Do not retry further.

## Fallback Model

A fallback model is a cheaper or more available model that meets the same capability floor. Choose it before the failure happens and store the choice in the pipeline contract.

| Primary | Typical fallback | Trade-off |
| --- | --- | --- |
| A high-intelligence reasoning model | A medium-intelligence model | Faster, cheaper, less rigorous on hard reasoning |
| A vision model | A text-only model with `multimodal_preprocess` | Lower quality on image-specific tasks |
| A premium model | A mid-tier model from the same provider | Cheaper, similar style, possibly weaker on edge cases |
| An integrated model | A gateway that uses a different model | More configuration, possibly different cost |

## Retry Mechanics

- Exponential backoff with jitter: start at 500ms, double, cap at 30s, jitter up to 250ms.
- Honor `Retry-After` when present.
- Use `idempotency_key` on chat completions to avoid duplicates on retry.
- Three attempts is the default. Five is the absolute cap.
- Time-box to 60 seconds total. Beyond that, the user has lost patience.

## Cost Discipline

Before each retry, estimate the cost. If the retry's expected cost exceeds the user's cap or the cost of failing, surface the error instead. A retry that consumes the user's balance without delivering value is a worse failure than the original error.

## Validation

- The retry (or fallback) succeeded.
- The trace ID is preserved.
- The retry attempts are recorded in the trace.
- The final report names the primary, the fallback used (if any), and the cost.
- The user's work is not duplicated.

## Escalation

- All retries failed: load `references/observability/situations/diagnose-degradation.md` to see if the failure is systemic.
- The fallback model is also failing: the failure is not model-specific. Inspect the gateway, the request, and the account state.
- The cost of retries is approaching the user's cap: stop. Surface the partial result and ask the user.

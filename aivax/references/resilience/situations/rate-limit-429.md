# Rate Limit (429)

Use when any AIVAX endpoint returns 429 Too Many Requests. The patterns apply to inference, generation, RAG, rerank, batch, and account endpoints.

## Objective

Get past the rate limit without burning the account balance, without retrying in a way that makes the limit worse, and without losing the user's work.

## Preconditions

- A `429` response; inspect either a standard `error` string or an OpenAI-compatible error object. `Retry-After` may be absent.

## Signals

- 429 status code.
- Recent request volume approaching a limit documented in [Plans and limits](https://docs.aivax.net/docs/limits); `Custom` limits require account-specific confirmation. Model listings and billing usage are not live rate-limit counters.
- Repeated 429s for the same model or endpoint across multiple requests.

## Decision Tree

1. Did the response include a `Retry-After` header? If yes, sleep for that duration and retry once. Do not retry earlier.
2. If no `Retry-After`, was the quota a per-minute request limit or a token-rate limit?
   - Per-minute request: back off exponentially with jitter, starting at 1s, doubling, capping at 30s.
   - Token-rate: reduce the payload (shorter context, smaller `top_n`, smaller `top` in `/api/v1/query`) before retrying.
3. Are three or more retries failing with the same class? Open the circuit breaker. Do not retry further. Switch to a fallback or surface the error.
4. Is the work in a batch or long-running job? Pause the job, wait, then resume. Do not start a new job while the old one is being throttled.

## Actions

1. Capture the 429 response exactly: status, body, headers (especially `Retry-After` and any quota headers).
2. Use balance and usage only for cost and plan context; they do not reveal the active rate-limit window.
3. Determine the applicable request or token-rate limit from the endpoint contract and request history. The generic rate-limit message may not name it; do not guess a quota or reset time. Storage exhaustion is a `402`, not this retry path.
4. Decide the retry strategy from the decision tree above.
5. If the work is large, prefer deferring over retrying (queue for batch, lower the request rate, downsize the payload).
6. After a retry succeeds, keep the request pace reduced; one success does not prove the whole rate-limit window has cleared.
7. If three retries fail, open the circuit breaker. Document the failure in the trace.

## Validation

- The retry (or fallback) succeeded within the time budget.
- The trace ID is preserved.
- The user's work is not duplicated.
- The observed response and retry timing are recorded without inventing quota counters.
- The circuit breaker is documented in the final report.

## Limits

- Do not assume changing the model, gateway, or API key resets a shared account limit.
- 429s often come in bursts. A retry that succeeds once may be followed by another 429. Plan for it.
- Rate-limit windows vary by operation. Subscription allowance exhaustion is separate and can lead to normal billing; it is not evidence of a `429` reset window.

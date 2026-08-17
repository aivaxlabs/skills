# Rate Limit (429)

Use when any AIVAX endpoint returns 429 Too Many Requests. The patterns apply to inference, generation, RAG, rerank, batch, and account endpoints.

## Objective

Get past the rate limit without burning the account balance, without retrying in a way that makes the limit worse, and without losing the user's work.

## Preconditions

- A 429 response with a JSON body that includes an error class and possibly a `Retry-After` header.

## Signals

- 429 status code.
- Quota counters near a plan limit (visible in `aivax_list_models` or `GET /api/v1/information/usage`).
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
2. Inspect the current quota state with `GET /api/v1/information/balance` and `GET /api/v1/information/usage?timeStart=<iso>&timeEnd=<iso>`.
3. Determine which quota was hit: per-minute request, token-rate, storage, or reranker request. The error message usually names the quota; if not, infer from the endpoint.
4. Decide the retry strategy from the decision tree above.
5. If the work is large, prefer deferring over retrying (queue for batch, lower the request rate, downsize the payload).
6. After the retry succeeds, re-check the quota state to confirm the new request is within limits.
7. If three retries fail, open the circuit breaker. Document the failure in the trace.

## Validation

- The retry (or fallback) succeeded within the time budget.
- The trace ID is preserved.
- The user's work is not duplicated.
- The quota report shows the new request landed within the limit.
- The circuit breaker is documented in the final report.

## Limits

- 429 is a quota problem, not a logic problem. Do not change the model, gateway, or schema to escape it; the new component will hit the same quota.
- 429s often come in bursts. A retry that succeeds once may be followed by another 429. Plan for it.
- Some quotas (token-rate) reset quickly; others (subscription reserve) reset on a longer window. Match the retry to the window.

# Transient Failure

Use for an unavailable dependency (`503` with `Retry-After`), a safe bounded retry after an unexpected AIVAX failure (`500`), or an evidenced temporary transport failure.

## Objective

Recover from a temporary failure without losing the user's work, without spending more than necessary, and without retrying in a way that compounds the failure.

## Preconditions

- The actual status and error envelope have been classified; a generic message alone does not establish a transient failure.
- The retry is safe for the operation and its possible partial effects. Dependency timeouts normally surface as `503`, not necessarily `408` or `504`.

## Decision Tree

1. Is the status a client error? Do not treat an unchanged `400` as retryable, even if a dependency's message says it is temporarily unavailable, and even when the cause is an unreadable request or dropped connection on your side (also a `400`). A zero data retention restriction with no eligible provider is a `400` too; change the model or the setting. Load `references/platform-rules/error-handling.md`; route `429` to the rate-limit scenario.
2. Could the call have partially applied? Inspect the resource before any retry. See `situations/idempotent-retry.md` and `situations/partial-failure.md`.
3. Did the response include `Retry-After`? Wait for that interval before a safe retry. Otherwise back off exponentially with jitter, starting at 500ms, doubling, capping at 30s.
4. After three failures with the same class, open the circuit breaker. Switch to an approved fallback or surface the error. A `500` does not guarantee that another attempt will succeed.

## Actions

1. Preserve the exact status, error code, and short message.
2. Capture the request ID for support and trace correlation.
3. Inspect the affected resource (if the call was a mutation) before deciding to retry.
4. Apply the retry strategy from the decision tree.
5. Re-validate through the same path the user will use.

## Validation

- The retry (or fallback) succeeded.
- The trace ID is preserved.
- The mutation's actual state was verified (no duplicates, no missing fields).
- The retry attempts are recorded in the trace.

## Limits

- Do not retry more than five times. A retry storm is a worse failure than the original error.
- Do not retry if the error message indicates a permanent class (validation, schema, plan limit). Those need a fix, not a retry.
- For mutations, retry only when the call is idempotent or the failure is guaranteed to be a no-op. Otherwise inspect first.

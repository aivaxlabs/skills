# Transient Failure

Use when an AIVAX endpoint returns a transient error: 5xx, 408, 504, or any error class the platform flags as temporary.

## Objective

Recover from a temporary failure without losing the user's work, without spending more than necessary, and without retrying in a way that compounds the failure.

## Preconditions

- A 5xx, 408, 504, or another clearly temporary error class.
- The error message or AIVAX docs flag the error as transient, or the agent has prior evidence that the same call succeeded recently.

## Decision Tree

1. Is the error class clearly transient? If not, do not retry — load `references/platform-rules/error-handling.md` and classify again.
2. Did the response include a `Retry-After` header? If yes, sleep for that duration and retry once.
3. If no `Retry-After`, back off exponentially with jitter, starting at 500ms, doubling, capping at 30s. Three attempts is usually enough.
4. After three failures with the same class, open the circuit breaker. Switch to a fallback or surface the error.
5. If the call is a mutation that may have partially applied, do not retry. Inspect the resource to determine actual state. See `situations/partial-failure.md`.

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

# Run And Inspect Results

Use when executing an Agentic Test and collecting its result, retained conversation, usage, error, and cost.

## Choose Persisted Or Ephemeral

- Use a persisted run when history, scheduling, cancellation, or later comparison matters.
- Use ephemeral SSE for one direct evaluation whose progress must be streamed and does not need to be retained as an Agentic Test run.

Both modes call paid models. Establish a reasonable turn budget before execution. One test run is one simulated conversation, not a batch of inputs.

## Persisted Procedure

```text
1. Verify the definition
   GET /api/v1/agentic-tests/{test-id}

2. Queue exactly one run (no request body)
   POST /api/v1/agentic-tests/{test-id}/runs

3. Save data.id from the 202 response as run-id

4. Poll one run, or list with bounded pagination
   GET /api/v1/agentic-tests/{test-id}/runs/{run-id}
   GET /api/v1/agentic-tests/{test-id}/runs?state=running&limit=50&offset=0

5. Stop polling at succeeded, failed, or cancelled

6. Inspect data.result, data.conversation, data.cost, and data.error
```

Do not call `/run`; the implemented queue endpoint is plural `/runs`. Do not send run metadata in the queue request. `metadata` and `external_user_id` are copied from the saved test definition.

## Interpret The Result

Keep these dimensions separate:

- `state`: operational lifecycle — `pending`, `running`, `succeeded`, `failed`, or `cancelled`.
- `result.outcome`: behavioral result — `success`, `loss`, or `incomplete`.
- `score`: final normalized conversation trajectory (`conversation_delta`), not an average over inputs.
- `error`: safe operational failure text, usually relevant when state is `failed`.
- `conversation`: timestamped `user`, `assistant`, and `judge` entries with per-message token usage and cost.
- `cost`: total billed cost attributed to the run.

A `succeeded` run may have `result.outcome: loss` or `incomplete`; succeeded only means the runner completed without an execution error.

Expected final reasons include:

- `baseline_reached`: judge score reached `base_threshold`.
- `loss_threshold_persisted`: the low trajectory persisted long enough to be considered lost.
- `simulated_user_ended_conversation`: the simulated user exited without success or loss.
- `max_turns_reached`: the turn budget ended without a terminal judge outcome.

## Ephemeral SSE Procedure

```http
POST /api/v1/generations/agentic-tests
Content-Type: application/json
Accept: text/event-stream
```

Use the request schema in `../SKILL.md`. Consume events until `chat.validation.end` or a terminal `unhandled_error`. Relevant event families are:

- `chat.user_message.*`
- `chat.assistant_message.*`
- `chat.judge.*`
- `usage_updated`
- `chat.validation.end`
- `unhandled_error`

The direct evaluation is not available later through the persisted run endpoints. If auditability matters, capture the SSE stream on the client or use a persisted test.

## Cancellation

```http
POST /api/v1/agentic-tests/{test-id}/runs/{run-id}/cancel
```

The endpoint has no body. Pending work is cancelled immediately; running work is cooperatively stopped. Calling it for a terminal run leaves that run unchanged. Already completed model calls remain billable.

## Failure Handling

- `400`: correct the model/gateway, goal, initial messages, profile, thresholds, sampling, or cron values; do not retry unchanged.
- `402` on ephemeral evaluation: stop and report insufficient operating balance.
- Persisted `failed` with insufficient-balance error: stop; do not repeatedly requeue.
- Retryable provider events in SSE may be retried internally (`will_retry: true`). Do not start a duplicate evaluation while the existing stream is active.
- A long run is not evidence of a hang while its state remains `running`; use bounded polling and cancel only under an agreed limit.

## Validation

Report the test ID, run ID or ephemeral mode, operational state, behavioral outcome, final score when present, reason, turns, cost, and any error. Quote retained judge reasoning when explaining why a run passed or failed.

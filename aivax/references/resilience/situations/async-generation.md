# Async Generation

Use when a generation should run in the background instead of holding the connection: slow work, work that can wait, or work that should take the cheapest provider. The request body is queued, replayed later against the same route, and its stored response is fetched or announced by webhook.

## Objective

Queue the call, retrieve its result without running it twice, and clean up failed or abandoned generations.

## Preconditions

- A private API key; public keys cannot queue (`400`). Listing and viewing need `ResourcesRead`; cancelling and deleting need `ResourcesWrite`.
- The account setting `parameters.asyncGenerationsEnabled` is on (the default). When off, queueing returns `403` and queued generations are kept but not executed. See `references/account/`.
- The call does not stream: `async` with `stream: true` is a `400`.

## Decision Tree

1. Many records sharing one workflow, schema, or validation? Use `references/batch/`. A few independent calls, or one slow call? Use `async`.
2. Does the user need the answer now? Call synchronously.
3. Does the route accept `async`? Chat completions, `/v1/responses`, `/v1/messages`, and under `/api/v1/generations/`: speech, audio transcription, images, classify, segment, summarize, translate, extract, rerank, decisions, and validations (an SSE route, so the stored response is the event stream text); plus `/api/v1/web/search` and `/api/v1/web/fetch`. Media description, instruction refinement, and skill teaching do not.
4. Must the result reach another system? Add `webhooks`, and keep polling as the fallback.

## Construction

```json
{
  "model": "<model-or-gateway>",
  "messages": [{ "role": "user", "content": "Summarize the attached report." }],
  "async": {
    "tags": ["report-2026-11"],
    "webhooks": ["https://example.com/hooks/aivax"],
    "timeout": 600
  }
}
```

- `"async": true` queues with defaults. In the object form, `enabled` defaults to `true`, and `async_tags` is a top-level alias for `async.tags`. Any other `async` type is a `400`.
- `tags`: up to 16, 1–64 characters each. `webhooks`: up to 4 public `http` or `https` URLs. `timeout`: seconds per execution attempt, 10–3600, defaulting to the maximum. Request bodies have a size cap.
- The reply is `202` with `data.async_generation.id` and `status: "pending"`, not the generation result. Nothing is billed until it runs.

Read the result:

```text
GET /api/v1/async-generations?status=<status>&tags=<a,b>&limit=<n>&offset=<n>
GET /api/v1/async-generations/<id>
GET /api/v1/async-generations/<id>/response
```

- The list returns summaries without bodies, newest first; `tags` keeps generations carrying every tag.
- `GET <id>` adds `webhooks`, `timeoutSeconds`, the original `request`, and the stored `response` (JSON embedded; other content, such as audio, as base64).
- `GET <id>/response` returns the stored body with its original status code and content type, as if the call had been synchronous. It is `404` until a final response is stored.
- Poll with backoff or wait for the webhook; never poll in a tight loop.

## Execution Rules

- Statuses: `pending`, `processing`, `retrying`, `done`, `cancelled`. Compare them case-insensitively. `done` means finished, not successful: check `responseStatusCode`.
- The run replays the stored body with the key that queued it. Keep that key alive until the generation finishes; otherwise the stored result is `401` and is not retried.
- Retryable failures (`402`, `429`, `5xx`, an unavailable service) are retried automatically a few times with growing delays, then stored as `done` with the failing status. Other failures, such as a `400`, a timeout (`408`), or a response too large to store (`507`), are stored at once.
- A request that names an integrated model without a routing preset or `#` suffix is routed with `Cheapest` instead of `Balanced`, so special providers become eligible when the account allows them. A saved gateway with an explicit routing option keeps it. Send `routing_options.preset` when the preset matters.
- After a final result, the webhook is one `POST` with `{ "id", "status": "done" }`. It is not sent for cancelled generations, carries no result, and is not retried: fetch `GET <id>/response`. When the account has a hook key, it sends `X-Request-Nonce`; validate it (see `references/account/`).
- Request and response bodies are stored until purged, regardless of the conversation-logging setting. Do not queue payloads the account must not retain, and copy results you need to keep.

## Cancel And Delete

- `PUT /api/v1/async-generations/<id>/cancel`: for `pending`, `retrying`, or `done` with an error status. A running or successful generation returns `409`. Cancelled generations are deleted after one hour.
- `PUT /api/v1/async-generations/cancel?scope=pending|failed|all`: bulk cancellation. Require explicit approval.
- `DELETE /api/v1/async-generations/<id>`: deletes the generation and its stored response (`204`; `409` while running). Destructive; require explicit approval.

## Validation

- The queue call returned `202`, and the `id` and tags are recorded.
- The final generation is `done` with `responseStatusCode` below `400`, and `GET <id>/response` matches the route's normal contract.
- The report names the `id`, final status, and `responseCost`.

## Failure Modes

- The `202` was lost: list by your tag before re-sending, because every queued call runs and bills on its own. Give each logical operation a stable tag.
- Queueing returns `429`: the account queued too many generations in the window. Wait, or move the work to batch. See `situations/rate-limit-429.md`.
- A generation stays `retrying` or ends `done` with `402`: check the balance before cancelling or re-queueing. See `references/account/`.
- Nothing runs and no error appears: confirm `asyncGenerationsEnabled` and that the queuing key still exists.
- A service restart re-queues a generation that was running, so an attempt can run twice. Do not queue calls with side effects that must happen once.
- Repeated `5xx`: load `situations/transient-failure.md`; do not re-queue in a loop.

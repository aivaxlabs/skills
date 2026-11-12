# Observe message.completed

Use when a gateway worker must record the outcome of a finished inference — generated messages, errors, usage — in an external system.

## Objective

Receive the completion notification, validate it, and record it idempotently without expecting the response to influence the inference.

## Preconditions

- The worker endpoint receives the `message.completed` event with `messages`, `errors`, `fatal`, `usage`, `origin`, `externalUserId`, and `metadata`.
- `gatewayId` and `X-Request-Nonce` are validated before trusting the body.

## Decision Tree

1. Is the notification for a gateway this endpoint serves? If not, return `2xx` and ignore it.
2. Did the inference fail (`fatal: true`)? Record the fatal entry in `errors`; `usage` can be `null` and `messages` can hold only the input.
3. Are there non-fatal entries in `errors`? Record them as warnings: the inference finished, but a provider, moderation, or tool-argument problem occurred.
4. Must the record reach an external store reliably? Persist quickly and process asynchronously; AIVAX does not retry this notification.

## Construction

The event payload:

```json
{
  "name": "message.completed",
  "data": {
    "messages": [
      { "role": "user", "content": "Where is order A123?" },
      { "role": "assistant", "content": "Order A123 is scheduled for delivery tomorrow." }
    ],
    "errors": [],
    "fatal": false,
    "usage": {
      "prompt_tokens": 120,
      "completion_tokens": 18,
      "total_tokens": 138,
      "prompt_token_details": { "cached_tokens": 0, "audio_tokens": 0 },
      "cost": 0,
      "performance": {}
    },
    "origin": "ChatCompletionsApi",
    "externalUserId": "customer-123",
    "metadata": {}
  }
}
```

- `messages`: the OpenAI-format messages of the inference, including the generated ones; messages can carry optional per-message metadata such as `id` or `usage`. When the inference failed before finishing, it holds the input messages.
- `errors[]`: `{ "message": "...", "fatal": false }` entries. A fatal entry carries the API or provider error message, or a generic internal-error message.
- `fatal`: `true` when an error interrupted the inference.
- `usage`: token usage, `cost`, and `performance` timings of the inference, or `null` when it failed before finishing. Use these values as reported; do not derive prices from them.

The worker response is ignored. Return `2xx`.

## Validation

- The endpoint accepts the event and returns quickly; records are idempotent if the same inference is observed twice.
- A test inference through the gateway produces one record with the expected messages and usage.
- A forced failure produces a record with `fatal: true` and a fatal entry in `errors`.

## Failure Modes

- Returning a worker action or a non-OK status to change the inference: the inference already finished and the response is ignored.
- Expecting one notification per client request: an attempt AIVAX retries internally does not notify; the retry raises its own `message.completed`.
- Relying on it for billing reconciliation: usage data is a snapshot from the gateway; use `references/cost-monitoring/` for authoritative account spend.
- Logging full `messages` without need: they contain user content; store only what the policy allows.

## Limits

- Delivery is asynchronous and best-effort: AIVAX sends it once in the background, does not wait for it, and does not retry on failure.
- It fires for gateways with a worker URL, including requests sent with conversation logging disabled.

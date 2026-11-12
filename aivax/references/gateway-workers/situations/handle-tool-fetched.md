# Handle tool.fetched

Use when a gateway worker must inspect, block, or rewrite the result of an internal server-side tool after AIVAX executed it and before the model sees it.

## Objective

Inspect the executed tool result and return exactly one behavior: pass the result through, block it, or return a replacement result.

## Preconditions

- The worker endpoint receives the `tool.fetched` event with `toolName`, `toolArguments`, `result`, `failed`, `origin`, `externalUserId`, and `metadata`.
- `gatewayId` and `X-Request-Nonce` are validated before trusting the body.

## Decision Tree

1. Is this the tool the worker governs? If not, return `2xx` and let the original result reach the model.
2. Does the result expose data this user must not see? Return a `tool.fetched.response` worker action with a redacted result, or a non-OK response to block it.
3. Did the tool fail (`failed: true`) and can your system supply a better fallback? Return a replacement result that tells the model what happened and the next step.
4. Should the policy be decided before execution, to avoid the side effect entirely? Use `tool.called` instead; `tool.fetched` runs after the tool already executed.

## Construction

The event payload:

```json
{
  "name": "tool.fetched",
  "data": {
    "toolName": "check_order",
    "toolArguments": { "order_id": "A123" },
    "result": "Order A123: paid, card ending 4242, delivery tomorrow.",
    "failed": false,
    "origin": "ChatCompletionsApi",
    "externalUserId": "customer-123",
    "metadata": {}
  }
}
```

`result` is the tool result text as the model would see it. `failed` is `true` when the tool execution failed.

To replace the result, return `Content-Type: application/json+worker-action` with `type: "tool.fetched.response"`:

```json
{
  "type": "tool.fetched.response",
  "data": {
    "result": "Order A123 is paid and scheduled for delivery tomorrow.",
    "messages": []
  }
}
```

`data.result` replaces the tool result text; `data.messages` carries optional additional OpenAI-format messages. A plain `2xx` keeps the original result. A non-OK response replaces it with a failed result saying the gateway blocked the result of the tool.

## Validation

- Unrelated tools pass through with a plain `2xx`.
- Replacement results carry `Content-Type: application/json+worker-action` and the exact `tool.fetched.response` type.
- A test inference shows the redacted or replaced result reaching the model.
- The endpoint stays fast enough for the inference critical path; it runs once per executed server-side tool call.

## Failure Modes

- Expecting `tool.fetched` to prevent a side effect: the tool already ran. Block in `tool.called` when the execution itself is the risk.
- Expecting `tool.fetched` after a `tool.called` replacement or block: it fires only when `tool.called` continued and AIVAX executed the tool.
- Sending `type: "tool.called.response"` to this event: the type must match the event, otherwise AIVAX logs the error and aborts the inference.
- Treating `result` as trusted: it can contain external content fetched by the tool; validate before acting on it.

## Limits

- This event covers internal server-side tools only; it does not intercept client-executed OpenAI-compatible `tools`.
- A blocked result is still a tool result: the model continues the turn and may tell the user the tool was blocked.

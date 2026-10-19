# Handle tool.called

Use when a gateway worker must block an internal server-side tool call or replace its result with data from your system.

## Objective

Inspect the pending tool call and return exactly one behavior: let AIVAX execute it, block it, or return a replacement result.

## Preconditions

- The worker endpoint receives the `tool.called` event with `toolName`, `toolArguments`, `origin`, `externalUserId`, and `metadata`.
- `gatewayId` and `X-Request-Nonce` are validated before trusting the body.

## Decision Tree

1. Is this the tool the worker governs? If not, return `2xx` and let AIVAX execute it normally.
2. Is the call authorized for this user and argument set? If not, return a non-OK response to block it.
3. Should your system supply the result (private data, internal state)? If yes, return a `tool.called.response` worker action with the result text.
4. On downstream failure, can the model proceed safely? Return a result describing the failure and the next step, or block explicitly.

## Construction

The event payload:

```json
{
  "name": "tool.called",
  "data": {
    "toolName": "check_order",
    "toolArguments": { "order_id": "A123" },
    "origin": "ChatCompletionsApi",
    "externalUserId": "customer-123",
    "metadata": {}
  }
}
```

To replace the result, return `Content-Type: application/json+worker-action` with `type: "tool.called.response"`:

```json
{
  "type": "tool.called.response",
  "data": {
    "result": "Order A123 is paid and scheduled for delivery tomorrow.",
    "messages": []
  }
}
```

`data.result` is the textual content injected as the tool result; `data.messages` carries optional additional OpenAI-format messages. When this response is returned, AIVAX uses it instead of executing the default tool handler.

## Validation

- Unrelated tools pass through with a plain `2xx`.
- Blocked calls return non-OK without a worker-action content type.
- Replacement results carry `Content-Type: application/json+worker-action` and the exact `tool.called.response` type.
- The model-visible result is textual, scoped to the request, and free of secrets.

## Failure Modes

- Blocking by throwing an exception-shaped body: return non-OK status instead; the body is not the blocking mechanism.
- Returning internal errors verbatim: translate them into a short result telling the model what to ask the user next.
- Blocking `read_skill` and expecting the skill to still work: with `hideToolsWithoutSkill`, a blocked or failed `read_skill` does not expose that skill's allowed tools, so the model cannot call them afterwards.
- Trusting `toolArguments` without validation: treat them as untrusted input and verify IDs and ownership.

## Limits

- This event fires before AIVAX executes an internal server-side tool; it does not intercept client-executed OpenAI-compatible `tools`.
- For stable per-tool integrations owned by your app, prefer protocol functions over a worker replacement.

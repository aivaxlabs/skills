# Handle message.received

Use when a gateway worker must allow, stop, or rewrite the conversation context before the model call.

## Objective

Inspect the incoming message context and return exactly one behavior: continue unchanged, stop the event, or apply a context rewrite action.

## Preconditions

- The worker endpoint receives the `message.received` event with `messages`, `origin`, `externalUserId`, and `metadata`.
- `gatewayId` and `X-Request-Nonce` are validated before trusting the body.

## Decision Tree

1. Is the caller authorized for this gateway and user? If not, stop the event with a non-OK response (fail closed).
2. Must the context change (policy instruction, tenant rule, enrichment)? If not, return `2xx` without a worker-action content type and continue normally.
3. Which rewrite fits? `add-system` for instructions, `add-message` / `remove-message` for message edits, `clear` plus `add-message` to replace the user context, `add-tool` / `add-protocol-tool` for message-dependent tools, `add-mcp-source` for message-dependent MCP tools.
4. Is the enrichment optional? On worker-side failure, return `2xx` to continue (fail open) or non-OK to stop (fail closed), per the policy.

## Construction

The event payload:

```json
{
  "name": "message.received",
  "data": {
    "messages": [],
    "origin": "ChatCompletionsApi",
    "externalUserId": "customer-123",
    "metadata": {}
  }
}
```

To rewrite, return `Content-Type: application/json+worker-action` with `type: "message.received.response"`:

```json
{
  "type": "message.received.response",
  "data": {
    "rewrites": [{ "type": "add-system", "message": "Answer in formal English." }]
  }
}
```

Available rewrite actions: `clear` (`argument`: `messages`, `meta`, `system`, `tools`, `skills`, `all`, or omitted), `add-message` (`message`: OpenAI-compatible message), `remove-message` (`index`: zero-based), `add-system` (`message`: instruction text), `add-tool` (`tool`: OpenAI-compatible tool definition), `add-protocol-tool` (`tool`: protocol function definition), `add-mcp-source` (`source`: MCP source with `url`, `headers`, `name`, `cacheDuration`).

## Validation

- Unauthorized callers get a non-OK stop response, not a silent continue.
- The rewrite response carries `Content-Type: application/json+worker-action` and the exact `message.received.response` type.
- A test inference shows the rewritten context reaching the model.
- The endpoint stays fast enough for the inference critical path.

## Failure Modes

- Wrong content type on a rewrite: AIVAX treats it as a plain continue and the rewrite never applies.
- Trusting `externalUserId` or `metadata` as proof of permission: validate the nonce and enforce your own authorization rules.
- Slow enrichment endpoint: move it out of the worker or fail open explicitly.

## Limits

- For permanent MCP sources, configure MCP directly in the gateway instead of injecting per message.
- `event.data` shape varies by origin; validate fields before indexing into them.

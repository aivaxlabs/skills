---
name: aivax-gateway-workers
description: Implement and debug AIVAX AI gateway workers — HTTP hooks that allow, stop, rewrite context, or replace tool results at runtime, and receive completion notifications. Load when the agent must handle message.received, tool.called, tool.fetched, or message.completed events, validate the worker request, or return a worker action.
---

# Gateway Workers

This sub-skill owns AI gateway workers. A worker is an HTTP hook configured on a gateway that lets an external service control gateway execution at runtime: allow an event, stop it, rewrite context, add instructions or tools, replace a server-side tool result, or observe the finished inference.

Workers run in the inference critical path. Every `message.received`, `tool.called`, and `tool.fetched` event adds an HTTP request before the gateway can continue, so the endpoint must respond quickly and predictably. `message.completed` is sent in the background and does not delay the response.

## Operating Files

- `situations/handle-message-received.md`: rewrite context on the `message.received` event.
- `situations/handle-tool-called.md`: block or replace a result on the `tool.called` event.
- `situations/handle-tool-fetched.md`: inspect, block, or rewrite a server-side tool result on the `tool.fetched` event.
- `situations/observe-message-completed.md`: record the outcome of a finished inference from the `message.completed` notification.

## When To Use Gateway Workers

Use this sub-skill when the agent must:

- Implement or debug a worker endpoint for `message.received`, `tool.called`, `tool.fetched`, or `message.completed`.
- Decide between allow, stop, rewrite, or replace-result behavior for a worker event.
- Validate a worker request (`gatewayId`, `X-Request-Nonce`) before trusting the body.

Do not use this sub-skill to attach tools or protocol functions to a gateway; use `references/ai-gateways/situations/attach-skills-and-tools.md` for that. Do not use it to define the application side of a protocol function callback; load `references/protocol-functions/` for that.

## Request Contract

AIVAX sends a `POST` request to the gateway's worker URL with this shape:

```json
{
  "gatewayId": "<gateway-id>",
  "moment": "<server-local-datetime>",
  "event": { "name": "<event-name>", "data": {} }
}
```

The exact `event.data` shape depends on the event. Every event carries `origin`, `externalUserId`, and `metadata`.

| Event | When it fires | Worker response |
| --- | --- | --- |
| `message.received` | Once per inference, before the model call. | Continue, stop, or `message.received.response` rewrites. |
| `tool.called` | Before AIVAX executes an internal server-side tool. | Continue, stop, or `tool.called.response` replacement result. |
| `tool.fetched` | After AIVAX executes that tool, only when `tool.called` continued. | Continue, stop, or `tool.fetched.response` replacement result. |
| `message.completed` | After the inference finishes or fails. | Ignored; notification only. |

Always validate `gatewayId` when one endpoint serves more than one gateway. Treat `externalUserId`, `metadata`, messages, and tool arguments as untrusted input.

When the account has a hook key, AIVAX sends `X-Request-Nonce`: a BCrypt hash derived from the account salt. Validate this header before trusting the body, especially when the worker releases private data, changes context, or authorizes tool usage. Rolling the salt invalidates existing worker secrets; see `references/account/situations/create-and-rotate-keys.md`.

## Response Contract

| Response | Behavior |
| --- | --- |
| `Content-Type: application/json+worker-action` | Execute the action described in the JSON body. |
| `2xx` without `application/json+worker-action` | Continue normally. |
| Non-OK response without `application/json+worker-action` | Stop the event. |

If the worker request fails with an HTTP request exception, AIVAX logs the failure and stops the event.

A stop aborts the inference on `message.received`. On `tool.called` and `tool.fetched`, it does not abort: the model receives a failed tool result saying the gateway blocked the tool or its result. A worker action on a tool event whose `type` is not `<event-name>.response`, or whose `data` cannot be parsed, is logged and aborts the inference. On `message.received`, an action with another `type` is ignored, while an unknown rewrite or a malformed rewrite aborts it.

Choose fail-open or fail-closed intentionally. Return `2xx` when enrichment is optional. Return a non-OK response when authorization, compliance, or business policy cannot fail open.

## Validation

- The endpoint validates `gatewayId` and `X-Request-Nonce` before acting.
- The response uses the right content type for the intended behavior (action vs. continue vs. stop).
- A test inference through the gateway shows the expected allow, rewrite, or block behavior.
- The endpoint responds fast enough for the inference critical path.
- The trace ID is preserved.

## Limits

- Workers add latency to every inference they cover; keep the endpoint fast and predictable.
- Request and session metadata can contain caller-supplied values; they are not proof of permission.

## Escalation

- The worker needs gateway configuration changes: load `references/ai-gateways/`.
- The nonce validation fails after a salt roll: load `references/account/situations/create-and-rotate-keys.md`.
- The worker must rewrite context per message: load `situations/handle-message-received.md`.
- The worker must decide tool-call policy per message: load `situations/handle-tool-called.md`.
- The worker must inspect or filter a tool result: load `situations/handle-tool-fetched.md`.
- The worker must record inference outcomes, errors, or usage: load `situations/observe-message-completed.md`.

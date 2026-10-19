---
name: aivax-protocol-functions
description: Define and serve AIVAX protocol functions — server-side tools the model calls by name while AIVAX executes your HTTP callback. Load when the agent must define a protocol function, serve its callback or function source, or debug a protocol function call.
---

# Protocol Functions

This sub-skill owns the application side of AIVAX protocol functions. A protocol function is a server-side tool: the model sees the name, description, and JSON argument schema; AIVAX executes the hidden callback URL from the server and attaches the textual result to the conversation.

A protocol function has two contracts: the model contract (public name, description, schema that guide the tool call) and the callback contract (hidden URL and headers AIVAX uses after the model calls it). Implementation details stay out of the prompt only when your service validates the nonce, validates arguments, and applies its own authorization.

## Operating Files

- `situations/define-protocol-function.md`: write the model contract (name, description, schema, inline vs. source).
- `situations/serve-protocol-callback.md`: implement the callback endpoint (nonce, arguments, authorization, result text).

## When To Use Protocol Functions

Use this sub-skill when the agent must:

- Expose one or a few stable HTTP actions owned by your app (check an order, open a ticket, validate a coupon, register a lead).
- Write the function name, description, and `contentFormat` schema for inline `protocolFunctions` or a `protocolFunctionSources` listing endpoint.
- Implement or debug the callback that receives `function.name`, `function.content` (arguments), and `context`, and returns the result text.
- Decide between protocol functions, MCP, built-in tools, workers, or raw `tools`.

Do not use this sub-skill for AIVAX-maintained capabilities (web search, code execution, memory); those are built-in tools (see `references/text-inference/situations/inference-with-tools.md`). Do not use it for event-time policy or message rewriting; load `references/gateway-workers/` for that.

## Rule of Thumb

| Need | Prefer |
| --- | --- |
| One or a few stable HTTP callbacks owned by your app | Protocol functions |
| A larger tool catalog, standardized discovery, or an existing MCP server | MCP (`references/ai-gateways/situations/attach-skills-and-tools.md`) |
| Web search, URL reading, code execution, image generation, memory, or HTTP request tools maintained by AIVAX | Built-in tools (`references/text-inference/situations/inference-with-tools.md`) |
| Event-time policy, message rewriting, dynamic tool injection, or blocking a tool call before execution | Workers (`references/gateway-workers/`) |
| Provider-native tool definitions that your own client will execute | Raw `tools` in an OpenAI-compatible request |

## Callback Contract Summary

- The callback receives the function name, the model-supplied arguments in `function.content`, and a `context` object (`externalUserId`, `metadata`, `callSource`, `conversationToken`, `moment`).
- The `X-Request-Nonce` header carries the account hook-key proof when configured; it is not a `context` field. Validate it before trusting the body.
- `context.externalUserId` can be null; `context.metadata` holds caller-supplied string pairs (never secrets); `context.conversationToken` correlates calls but is not a unique callback ID or a dedup guarantee.
- Successful responses are textual and written for the model: data, warnings, and the next step. On recoverable errors (not found, bad field), return OK with a short human description and a workaround, not a raw stack trace.

## Validation

- The model calls the function for the right triggers and skips it for negative cases.
- The callback validates nonce, arguments, and user/tenant authorization before acting.
- Write operations have duplicate-operation controls; `conversationToken` alone is not proof an operation has not run.
- The result text is scoped, textual, and secret-free.
- The trace ID is preserved.

## Limits

- Function definitions consume reasoning tokens; keep the catalog to the smallest set that solves the use case.
- Sourced functions are cached for 10 minutes per account and source URL; a non-success listing response fails inference preparation.
- Function names must be valid JavaScript identifiers with at least three characters; names must stay unique across inline, sourced, MCP, built-in, and raw tools.

## Escalation

- The function must be attached to a gateway: load `references/ai-gateways/situations/attach-skills-and-tools.md`.
- The policy belongs at event time rather than call time: load `references/gateway-workers/`.
- The catalog is large or already MCP-shaped: expose MCP instead of multiplying protocol functions.
- The call hits rate-limit or transient failures: load `references/resilience/`.

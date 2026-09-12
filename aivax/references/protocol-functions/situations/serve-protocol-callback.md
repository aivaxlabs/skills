# Serve Protocol Callback

Use when the agent must implement or debug the HTTP callback AIVAX invokes after the model calls a protocol function.

## Objective

Validate the callback request (nonce, arguments, authorization), execute the action once, and return a short textual result the model can use.

## Preconditions

- The function name and `contentFormat` schema are defined on the gateway (inline or sourced).
- The account hook-key setup is known, so the endpoint knows whether to expect `X-Request-Nonce`.
- The caller identity source (`context.externalUserId`, `context.metadata`) and its nullability are understood.

## Decision Tree

1. Validate `X-Request-Nonce` when the account has a hook key. Reject unauthenticated callbacks before any side effect.
2. Validate arguments in `function.content` against the function schema, then enforce business authorization: look up `context.externalUserId`, check tenant and resource ownership. `tenant_id`, `callSource`, and `conversationToken` are not proof of permission.
3. Mutating action? Require explicit approval semantics and duplicate-operation controls of your own; `conversationToken` correlates calls but does not guarantee the operation has not run.
4. Success or recoverable error? Return OK with textual content either way: data plus next step on success; short human description plus workaround on not-found or bad-field errors.
5. Is the result model-ready? Data, warnings, and usage guidance only. No HTML, raw logs, internal messages, or secrets.

## Construction

The callback request carries the function name, the model-supplied arguments, and the execution context:

- `function.name`: the called function.
- `function.content`: the model-supplied arguments (validate against the schema).
- `context.externalUserId`: session or request user ID, or null when unidentified.
- `context.metadata`: caller-supplied string pairs (never put secrets here).
- `context.callSource`: inference origin (`WebChatClient`, `ChatCompletionsApi`, `IntegrationBot`).
- `context.conversationToken`: correlation token from `idempotency_key` when supplied; shared across calls, not a unique ID.
- `context.moment`: server-local timestamp of callback preparation.

Custom metadata stays nested under `context.metadata`; it is not merged into `context`.

## Validation

- Unauthenticated or unauthorized calls are rejected before side effects.
- Arguments match the schema and the business rules; anonymous calls to user-scoped functions are rejected, not silently granted.
- The response is textual, scoped, and secret-free, with a workaround on recoverable errors.
- The trace ID is preserved.

## Failure Modes

- Trusting metadata or `callSource` as authorization: enforce your own user and tenant checks.
- Returning raw errors or huge objects: the content enters the model context and degrades the next response. Summarize instead.
- Missing idempotency on writes: retries and multi-call turns execute the action twice. Add your own duplicate-operation control.

## Limits

- The nonce verifies the account hook key, not the individual user's permissions; it does not sign the body or prevent replay.
- Schemas limit structure but cannot replace business authorization or consistency checks.

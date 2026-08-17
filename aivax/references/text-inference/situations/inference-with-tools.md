# Inference With Tools

Use when an AIVAX chat completion must call an OpenAI-compatible function tool or an AIVAX built-in tool before returning a final answer.

## Objective

Give the model only the tools it needs, validate every requested action before execution, and feed tool results back in a format the model can use safely.

## Preconditions

- The task genuinely requires external data or an action beyond text generation.
- Each tool has a clear JSON Schema for its arguments and an accountable execution handler.
- Any action with side effects has an approval boundary and idempotency strategy.

## Decision Tree

1. Is a tool call necessary? Prefer direct reasoning when the answer is already in the messages or gateway context.
2. Is the capability available as an AIVAX built-in tool or through the gateway? Prefer configured reusable capability over duplicating a direct function definition.
3. Is the tool read-only or mutating? Mutating tools require explicit user approval at the action boundary.
4. Can arguments be validated deterministically? If not, tighten the function schema or avoid the tool.
5. Is the caller using a public key? Only OpenAI-compatible `tools` are available; built-in tools, skills, MCP sources, protocol functions, and Bash are stripped.

## Construction

Send function definitions in the OpenAI-compatible `tools` array:

```json
{
  "model": "<model-or-gateway>",
  "messages": [{ "role": "user", "content": "Find the order status for 123." }],
  "tools": [{
    "type": "function",
    "function": {
      "name": "get_order_status",
      "description": "Return current status for one order ID.",
      "parameters": {
        "type": "object",
        "additionalProperties": false,
        "required": ["orderId"],
        "properties": {
          "orderId": { "type": "string" }
        }
      }
    }
  }]
}
```

For each requested tool call:

1. Validate the function name against the offered tools.
2. Parse and validate arguments against the function schema.
3. Check authorization, approval, and idempotency before a mutating action.
4. Invoke the handler with the least required data.
5. Append the tool result to the conversation using the tool-call identifier returned by the model.
6. Call `/v1/chat/completions` again so the model can produce the final answer.

Use `builtin_tools` only for documented AIVAX built-in tools such as `WebSearch`, `AdvancedWebUsage`, `OpenUrl`, `Code`, `Request`, `Calendar`, and `Remember`. Verify supported tools and options with `aivax_search_context`.

## Validation

- Every invoked function was offered in the request and received schema-valid arguments.
- Read-only tool results are attributable to the correct request and user context.
- Mutating calls have explicit approval and avoid duplicate execution.
- The final answer reflects the tool result without exposing secrets or raw sensitive payloads.
- Tool-call errors return a safe, structured result to the model instead of silently fabricating an answer.

## Failure Modes

- Unknown function name or malformed arguments: do not invoke it; return a structured tool error and let the model recover or ask for clarification.
- Repeated calls after a transient failure: use an idempotency key and load `references/resilience/` before retrying.
- Tool output exceeds useful context: return a minimal structured summary or reference, then configure appropriate gateway `toolContextCount` if needed.
- The model uses the wrong tool: narrow descriptions and schemas, or adjust the gateway's attached tool surface.

## Escalation

- Need to attach tools, MCP sources, or protocol functions to a gateway: load `references/ai-gateways/situations/attach-skills-and-tools.md`.
- Need retry, idempotency, or partial-failure policy: load `references/resilience/`.
- Need trace analysis: load `references/observability/`.

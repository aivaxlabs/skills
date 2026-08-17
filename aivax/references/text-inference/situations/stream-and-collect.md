# Stream And Collect

Use when the agent is calling `POST /v1/chat/completions` with `stream: true` and must process the SSE response.

## Objective

Stream a chat completion to the user or to a downstream system, accumulate the assistant's text, parse the final usage block, and detect tool calls and finish reasons — without losing data and without breaking the trace.

## Preconditions

- The model and gateway are chosen.
- The downstream consumer can handle incremental text, or the agent is buffering it locally.
- The agent has a way to surface progress (logs, UI, or another stream).

## Decision Tree

1. Is the user waiting on a screen? Stream.
2. Is the agent going to validate the output before showing it? Buffer and validate, then surface. Buffering with a short initial delay (e.g. 200ms) is sometimes a good compromise for short responses.
3. Is the consumer a pipeline that does not need incremental text? Buffer and parse. Streaming adds complexity with no value.
4. Does the consumer need tool calls as they happen? Stream and parse `tool_calls` deltas. Most tool-calling pipelines stream because the tool call appears before the assistant's natural-language text.
5. Does the conversation have a `metadata.trace_id`? Pass it on the request and confirm it surfaces in the final conversation record.

## Stream Parsing

The SSE stream emits events with `data:` prefixes. Each event is a JSON object that is a partial chat-completion chunk. Parse each event, accumulate:

- `choices[i].delta.content`: append to the assistant text buffer.
- `choices[i].delta.tool_calls`: append to a tool-call buffer. The first event may have only `id`; later events add `function.name` and `function.arguments`.
- `choices[i].finish_reason`: capture the last non-null value (`stop`, `tool_calls`, `length`, `content_filter`).
- The final event has `[DONE]` as the data payload and carries the `usage` block when the server reports it.

A common pattern:

```text
buffer_text = ""
buffer_tool_calls = []
for event in stream:
  if event is "[DONE]":
    finalize(usage = event.usage, finish_reason = last_finish_reason)
    return
  for choice in event.choices:
    if choice.delta.content:
      buffer_text += choice.delta.content
      surface(buffer_text)
    if choice.delta.tool_calls:
      buffer_tool_calls = merge(buffer_tool_calls, choice.delta.tool_calls)
    last_finish_reason = choice.finish_reason or last_finish_reason
```

## Optional Headers

- `Sse-Stream-Options: no-ping` — disable periodic keep-alive events when the consumer does not need them.

## Validation

- The accumulated text is non-empty and matches the expected length.
- Every tool call has a non-empty `function.name` and a parseable `function.arguments` (JSON).
- The `usage` block has `prompt_tokens`, `completion_tokens`, and `total_tokens` (or a normalized equivalent).
- The `finish_reason` is one of the expected values.
- The trace ID is recorded in the conversation.

## Failure Modes

- **Truncated stream**: a network drop or a 5xx can cut the stream mid-response. Treat the partial text as candidate output, not final output. Validate or re-request.
- **No usage block**: some streams do not include `usage`. Estimate it from the prompt and the partial text length, and tag the estimate.
- **Empty `choices`**: the response is a non-streaming object. Switch to the non-streaming parser.
- **Tool call without `id`**: the server may stream a tool call delta without an ID if the model started one in a previous chunk. Carry the ID forward from the previous chunk.

## Escalation

- The stream is consistently slow: load `references/observability/situations/diagnose-degradation.md`.
- The cost per call is higher than expected: load `references/cost-monitoring/situations/investigate-cost-spike.md` and inspect `usage.prompt_tokens_details.cached_tokens` to see if caching is missing.
- The model is producing partial JSON inside a structured-output call: load `situations/structured-output.md` and consider JSON Healing.

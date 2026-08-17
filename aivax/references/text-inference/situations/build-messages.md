# Build Messages

Use when the agent must construct the `messages` array for a chat completion. The shape is OpenAI-compatible; the discipline is AIVAX-specific.

## Objective

Produce a messages array that the model can act on correctly, that fits the context window, and that preserves any conversation history, tool results, and multimodal parts without losing signal.

## Preconditions

- The task and the user-facing role are clear.
- The model is known (see `situations/choose-model.md`).
- Any required multimodal parts are prepared (see `references/multimodal/`).
- Any required tool definitions are prepared (see `situations/inference-with-tools.md`).

## Decision Tree

1. Is the task a single-turn prompt with no history? Use a `messages` array with one `user` message. A `system` message is optional but recommended for tone and constraints.
2. Is the task a multi-turn conversation? Use a `messages` array with the full history, oldest first. Do not collapse the history unless the model context is too small; collapsing loses signal.
3. Is the task a tool call? Include the assistant message that produced the tool call, the tool result message(s), and the next user or assistant message. Do not interleave unrelated text.
4. Is the task structured output? Add a `system` message that names the desired schema. The schema also lives in `response_format` or `response_schema`, but the system message is the place to tell the model what the JSON is for.
5. Is the task sensitive (PII, regulated content, customer data)? Avoid putting it in the system message if a system message would be cached for a different user. Keep the sensitive content in the user message.

## Message Construction Rules

- **Order**: oldest first. The most recent user message is the last element.
- **Roles**: `system`, `user`, `assistant`, `tool`, and (for some reasoning models) a reasoning part inside `assistant`. Do not invent roles.
- **Tool messages**: the `tool` role message has `tool_call_id` matching the assistant's `tool_calls[i].id`, and `content` is the tool result. Do not include unrelated text in a tool message.
- **Multimodal content**: `content` may be a string or an array of parts. Each part has a `type` (`text`, `image_url`, `video_url`, `input_audio`, `file`). See `references/multimodal/`.
- **Length**: keep each message under the model context window. For long contexts, prefer truncation with a clear summary over silent drops.
- **Caching**: keep the system message and any long prefix identical across calls when input caching matters. The cached input cost is typically 5x to 10x cheaper.

## Content Hygiene

- Do not include secrets, credentials, payment data, or large payloads in `metadata`. `metadata` is stored with the conversation.
- Do not include media that the model cannot handle. Verify the model supports the modality.
- Do not paste full transcripts, hidden reasoning, or private user data into final responses unless the user explicitly needs that exact data.

## Validation

- The `messages` array parses as valid JSON.
- Every `tool_call_id` has a matching `tool` message.
- The total token estimate (rough: 4 characters per token) is within the model's context window minus the expected completion length.
- The model returns a valid chat completion with a non-empty `choices[0].message`.

## Escalation

- The context is too large for the model: split the task or switch to a model with a larger window.
- The conversation history is contaminated with an earlier failure: clean it before sending, and document the cleaning.
- The tool call results are too large for a single message: summarize the result and pass the summary in a `user` message instead of the raw tool message.

---
name: aivax-text-inference
description: Run text inference on AIVAX — model selection, chat-completion messages, streaming, tool calls, function calling, structured output, retries, and fallbacks. Load when the agent is generating, transforming, or analyzing text with a hosted or BYOK model through AIVAX.
---

# Text Inference

This sub-skill owns everything about generating text with AIVAX models: choosing a model, building messages, streaming, calling tools, parsing structured output, and handling errors. The OpenAI-compatible endpoint is the surface; the contract is what the agent should expect and produce.

## Operating Files

- `model-catalog.md`: how to query and interpret `aivax_list_models` and the public model listing.
- `situations/choose-model.md`: pick a model for a given task, cost, latency, quality, and modality floor.
- `situations/build-messages.md`: compose a messages array for a chat completion.
- `situations/stream-and-collect.md`: handle SSE streaming, partial outputs, and finalization.
- `situations/inference-with-tools.md`: tool calling, function definitions, tool result handling.
- `situations/structured-output.md`: response schema, JSON Healing, response_format, json_only.
- `situations/fallback-and-retry.md`: model swap, exponential backoff, and cost-aware retries.

## When To Use Text Inference

Use this sub-skill when the agent needs to:

- Generate text from a prompt, a messages array, or a chat history.
- Call a hosted model directly (no gateway) for one-off or experimental work.
- Build a chat completion through an AI gateway (the gateway is the contract; this sub-skill owns the model and message contract).
- Run inference with tool calls, function calling, or structured output.
- Stream a response to the user or another system.

Do not use this sub-skill for:

- Generating images (load `references/image-generation/`).
- Generating or transcribing audio (load `references/speech/`).
- Realtime bidirectional voice (load `references/voice-realtime/`).
- Classifying or segmenting text (load `references/text-tools/`).
- Retrieving from a knowledge base (load `references/rag/`).
- Configuring a gateway (load `references/ai-gateways/`).

## Endpoint

`POST /v1/chat/completions` (also exposed as `/api/v1/chat/completions`).

The endpoint accepts OpenAI-compatible payloads plus AIVAX extensions:

- `model`: an integrated model name, an AI gateway UUID, or an AI gateway slug (private keys only).
- `messages`: an OpenAI-compatible messages array. Multimodal content parts are described in `references/multimodal/`.
- `prompt`: alternative to `messages` for simple text completions.
- `stream`: `true` to receive SSE; `false` for a single response.
- `temperature`, `top_p`, `top_k`, `seed`, `max_completion_tokens`, `reasoning_effort`, `stop`: standard sampling parameters.
- `tools`: an array of OpenAI-compatible tool definitions (function calling).
- `builtin_tools`: AIVAX-specific on-demand built-in tools (`WebSearch`, `AdvancedWebUsage`, `OpenUrl`, `Code`, `Request`, `Calendar`, `Remember`).
- `response_format` with `json_schema`: provider-compatible structured output.
- `response_schema`: AIVAX-specific structured output with JSON Healing.
- `json_only`: when the caller expects only the generated JSON, not the chat-completion envelope.
- `multimodal_preprocess`: AIVAX-specific flag to convert media into a text description before the main call.
- `idempotency_key`: string up to 128 characters. Reuse the same value to update the same stored conversation record.
- `metadata`: a JSON object of string key/value pairs, stored with the conversation record for correlation.
- `tool_invocation_explanations`: when supported, expose tool-call explanations in the response.

## Direct Call vs. Gateway

Decide explicitly:

- **Direct call** to a hosted or BYOK model: use when the configuration is not reused, the prompt is one-off, or the agent is exploring. `model` is a model name from `aivax_list_models` or a provider slug.
- **AI gateway**: use when the configuration (model, instructions, RAG, tools, skills, moderation) is reused across many calls or multiple users. The gateway UUID or slug goes into `model`. The gateway owns everything else.

When the user already maintains a gateway, the gateway is almost always the right choice. Direct calls are for prototyping or for cases the gateway cannot express.

## Modality

The endpoint accepts multimodal content parts: `text`, `image_url`, `video_url`, `input_audio`, and `file`. The model must support the modality unless `multimodal_preprocess` is used to convert media into a text description first. See `references/multimodal/`.

## Cost Awareness

Every inference call consumes balance, request quota, and possibly token-rate quota. A model swap to a cheaper or more expensive tier can change cost by 2x to 10x. Use `aivax_list_models` to compare pricing before recommending a swap. See `references/cost-monitoring/`.

## Validation

- The response is a chat completion with a `choices[0].message` and (for non-streaming) a `usage` object.
- For streaming, the agent collects the SSE events, accumulates the assistant text, and parses the final `usage` block.
- For tool calls, the agent validates the function name and arguments before invoking the tool.
- For structured output, the agent validates the JSON against the schema. JSON Healing may have already corrected the output; do not assume the model produced valid JSON on the first try.
- The trace ID propagates through `metadata` for observability. See `references/composition/situations/cross-skill-observability.md`.

## Limits

- The Free plan has a context cap. Confirm the selected model and prompt fit before sending.
- Some models support only a subset of the parameters. Verify with `aivax_list_models` or `aivax_search_context`.
- Public keys (`pk-aiv-`) cannot call integrated models directly. They call only the full AI Gateway UUID, with stripped tool surfaces.
- Tool calls inside a public-key call are limited to `tools` (OpenAI-compatible). `builtin_tools`, MCP sources, protocol functions, skills, and Bash are stripped.

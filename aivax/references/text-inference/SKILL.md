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
- `reasoning_effort`: take the value from the model's published list, `_details.provider_model.model_information.reasoningEfforts` in `/v1/models` (`modelInformation.reasoningEfforts` in `GET /api/v1/information/models.json`), in the order the source reports it. An empty list means no list is published, not that reasoning is unsupported. Some models reject the parameter outright; gateways fail validation when it is set for them.
- `tools`: an array of OpenAI-compatible tool definitions (function calling).
- `tool_choice`: optional tool-selection behavior forwarded to compatible gateways.
- `builtin_tools`: AIVAX-specific on-demand built-in tools (`WebSearch`, `Code`, `OpenUrl`, `Remember`, `ImageGeneration`, `XPostsSearch`, `GenerateDocument`, `GenerateWebPage`, `Request`, `DateTime`). Include `DateTime` in `tools` to expose `get_date_time`.
- `time_zone`: optional IANA time zone, such as `America/Sao_Paulo`, used by `get_date_time`, the scheduling tools, and the Bash tool for this request only. It overrides the AI gateway `parameters.timeZone`, which defaults to `America/Sao_Paulo`. Independent of the browser time zone. Invalid identifiers are rejected. The old `builtin_tools.options.dateTimeTimeZone` option no longer exists.
- `user`: optional external user identifier attached to the gateway inference context.
- `extra_body`: provider-specific request fields, supported only with BYOK AI Gateways.
- `routing_options`: private-key-only integrated-model routing override, with `preset` (`Balanced`, `Cheapest`, `Fastest`, or `Quality`) and ordered `allowed_providers` tags. Omitted fields inherit the gateway's `routingOption` and `allowedProviders`; defaults are `Balanced` and `["*"]`. Matching is case-insensitive; empty lists or blank tags are rejected. Include `"*"` last only when any-provider fallback is intended; otherwise the request fails if no listed provider is available. It does not change the saved gateway.
- `routing_preset`: deprecated preset override; `routing_options.preset` takes precedence when both are supplied. Public keys cannot send either routing extension.
- `rendering_mode`: set to `textual_blocks` to normalize reasoning into textual think blocks.
- `response_format` with `json_schema`: provider-compatible structured output.
- `response_schema`: AIVAX-specific structured output with JSON Healing.
- `json_only`: when the caller expects only the generated JSON, not the chat-completion envelope.
- `multimodal_resolver`: per-media engines (`imageEngine`, `audioEngine`, `videoEngine`, `fileEngine`) that convert selected inputs into text. It takes precedence over the deprecated `multimodal_preprocess` flags and the gateway's resolver configuration. See `references/multimodal/`.
- `idempotency_key`: AIVAX-specific optional non-empty string (not part of the OpenAI spec; check [Limits](https://docs.aivax.net/docs/limits) for its size constraint), mapped to `ConversationToken` for stored-conversation correlation. Reusing it may update the same stored conversation record, but does not deduplicate inference execution; each request still executes.
- `metadata`: a JSON object of string key/value pairs, stored with the conversation record for correlation.
- `allow_logging`: optional boolean, default `true`. Send `false` to keep this request out of the account conversation logs; it then cannot be found in `references/observability/` afterwards, so use it only when that loss is intended. It does not change provider-side retention (see Limits). Private keys only.
- `tool_invocation_explanations`: when supported, expose tool-call explanations in the response.

These fields are implemented in `OpenIndexer.Alpha2/Api/Modules/OpenAi.cs`. The built-in names come from `OpenIndexer.Alpha2/Inference/ReasoningChainResolverOptions.cs`.

For integrated models, the response includes the serving provider's routing tag in `provider`, next to `model`, including streamed chunks. BYOK gateways return `provider: null`. Use this observed value rather than inferring the provider from the requested model name.

## Minimal Tool-Call Loop

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
        "properties": { "orderId": { "type": "string" } }
      }
    }
  }]
}
```

1. Validate each requested function name against the offered `tools` and validate arguments against the function schema.
2. Execute the handler with the least required data.
3. Resend the tool result as a `tool` message carrying the original tool-call identifier.
4. Call `/v1/chat/completions` again with the appended result so the model produces the final answer. See `situations/inference-with-tools.md` for the full pattern.

## Direct Call vs. Gateway

Decide explicitly:

- **Direct call** to a hosted or BYOK model: use when the configuration is not reused, the prompt is one-off, or the agent is exploring. `model` is a model name from `aivax_list_models` or a provider slug.
- **AI gateway**: use when the configuration (model, instructions, RAG, tools, skills, moderation) is reused across many calls or multiple users. The gateway UUID or slug goes into `model`. The gateway owns everything else.

When the user already maintains a gateway, the gateway is almost always the right choice. Direct calls are for prototyping or for cases the gateway cannot express.

## Modality

The endpoint accepts multimodal content parts: `text`, `image_url`, `video_url`, `input_audio`, and `file`. The model must support the modality unless `multimodal_resolver` converts that media type into text first. See `references/multimodal/`.

## Cost Awareness

Every inference call consumes balance, request quota, and possibly token-rate quota. Check [Pricing](https://docs.aivax.net/docs/pricing) and [Limits](https://docs.aivax.net/docs/limits) before recommending a model swap; do not assume fixed price ratios. See `references/cost-monitoring/`.

## Validation

- The response is a chat completion with a `choices[0].message` and (for non-streaming) a `usage` object.
- For streaming, the agent collects the SSE events, accumulates the assistant text, and parses the final `usage` block.
- For tool calls, the agent validates the function name and arguments before invoking the tool.
- For structured output, the agent validates the JSON against the schema. JSON Healing may have already corrected the output; do not assume the model produced valid JSON on the first try.
- The trace ID propagates through `metadata` for observability. See `references/composition/situations/cross-skill-observability.md`.

## Limits

- The Free plan has a context cap. Confirm the selected model and prompt fit before sending.
- Some models support only a subset of the parameters. Verify with `aivax_list_models` or `aivax_search_context`.
- The account option `parameters.zeroDataRetentionOnly` (set through `PUT /api/v1/accounts`, which replaces `parameters`: read first and preserve the rest) restricts integrated text inference to providers with zero data retention, before routing options are applied. A model with no such provider fails with a client error that names the restriction; choose another model or ask the account owner to change the setting. It does not affect BYOK gateways. Per-provider policy is in `technicalInfo.dataCollection` of `providers[]` in `models.json`.
- Public keys (`pk-aiv-`) cannot call integrated models directly. They call only the full AI Gateway UUID, with stripped tool surfaces.
- Tool calls inside a public-key call are limited to `tools` (OpenAI-compatible). `builtin_tools`, MCP sources, protocol functions, skills, and Bash are stripped.

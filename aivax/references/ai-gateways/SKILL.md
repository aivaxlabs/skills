---
name: aivax-ai-gateways
description: Design, configure, and safely edit AIVAX AI Gateways — the configured runtime that aggregates model, instructions, RAG, skills, tools, MCP sources, multimodal preprocessing, and OpenAI-compatible inference behavior. Load when the agent must create a new gateway, attach skills or tools, or change a gateway's parameters.
---

# AI Gateways

This sub-skill owns AIVAX AI Gateways. A gateway is the configured runtime for an assistant: it chooses the model, sets the system instruction, attaches RAG collections, exposes tools and skills, and exposes an OpenAI-compatible endpoint. Most production AIVAX work goes through a gateway.

## Operating Files

- `situations/design-gateway.md`: design a gateway from scratch.
- `situations/attach-skills-and-tools.md`: attach skills, MCP sources, protocol functions, and built-in functions to a gateway.
- `situations/edit-gateway-safely.md`: edit a gateway's parameters with the smallest safe change.

## When To Use AI Gateways

Use this sub-skill when the agent must:

- Create a new gateway for a new assistant or use case.
- Attach or detach a skill, an MCP source, a protocol function, or a built-in function.
- Edit a gateway's parameters (model, instruction, RAG settings, context, output limits).
- Inspect a gateway's current configuration and recent behavior.

Do not use this sub-skill for:

- A direct chat completion (load `references/text-inference/`).
- A RAG-only operation (load `references/rag/`).
- A skill-only operation (load `references/skill-development/`).

## Gateway Concept

A gateway has:

- `name`: a human-readable name.
- `parameters`: the configuration object. Most parameters are at the top level of `parameters`; some are nested (e.g. `moderationParameters`).
- `parameters.baseAddress`: `@integrated` for AIVAX managed models, or an external OpenAI-compatible endpoint.
- `parameters.modelName`: a model returned by `aivax_list_models` or a gateway slug (private keys only).
- `parameters.systemInstruction`: the base behavior of the gateway.
- `parameters.knowledgeCollections`: collection IDs queried before model call.
- `parameters.rerankerName`: the reranker for RAG. Default is `lexical`. See `references/rerankers/`.
- `parameters.skills`: flat list of AIVAX skill IDs. No native priority or primary skill concept.
- `parameters.mcpSources`, `parameters.protocolFunctions`, `parameters.protocolFunctionSources`: external capabilities.
- `parameters.builtinFunctionsOptions`: options for built-in web search, image generation, memory, and related tools.
- `parameters.sentinelOptions.enabledFunctions`: built-in tools exposed through Sentinel reasoning.
- `parameters.enableBash` and `parameters.bashOptions`: controlled virtual Bash behavior.
- `parameters.toolContextCount`: how many previous tool results stay in context.
- `parameters.contextMaximumSize` and `parameters.contextOverflowAction`: cap and handle long context.
- `parameters.moderationParameters`: safety thresholds.
- `parameters.enabledMultimodalFeatures`: input media AIVAX may resolve and forward.
- `parameters.queryStrategy`: `Plain`, `Concatenate`, `FullRewrite`, `UserRewrite`, or `QueryFunction`.
- `parameters.knowledgeBaseMaximumResults`: number of documents injected.
- `parameters.knowledgeBaseMinimumScore`: similarity cutoff from 0 to 1.
- `parameters.knowledgeUseReferences`: include referenced parent documents.
- `parameters.knowledgeUseMetaDescriptions`: include metadata descriptions in retrieved context.
- `parameters.temperature`, `parameters.topP`, `parameters.presencePenalty`, `parameters.frequencyPenalty`, `parameters.stop`, `parameters.maxCompletionTokens`, `parameters.reasoningEffort`, `parameters.verbosity`: sampling and output.
- `parameters.userPromptTemplate`, `parameters.assistantPrefill`, `parameters.includePrefillingInMessages`: prompt shaping.
- `parameters.systemInstructionsSources`: external text resources appended to the system prompt.

## Public vs Private Keys

Public keys (`pk-aiv-`) can call chat completions with the full AI Gateway UUID. Direct integrated-model calls and gateway slug lookup are disabled. MCP sources, protocol functions, built-in tools, Bash, skills, and sentinel options are stripped from the request. Only these parameters are accepted: `model`, `messages`, `prompt`, `temperature`, `top_p`, `top_k`, `seed`, `tools`, `reasoning_effort`, `max_completion_tokens`, and `stream`.

## Cost Awareness

A gateway is the dominant cost driver for most accounts. The model choice, the RAG settings, the tool surface, and the moderation level all affect cost. Load `references/cost-monitoring/` when investigating or optimizing.

## Validation

- The gateway view reflects the new configuration.
- A test conversation or inference smoke test produces the expected result.
- The linked chat clients, skills, and collections are still valid.
- Recent conversations do not show new errors or unexpected token growth.
- The trace ID is preserved.

## Limits

- `PATCH /api/v1/ai-gateways/<id>` shallow-merges top-level gateway fields and top-level `parameters` fields. Send only the fields that should change.
- Arrays are usually replacements during patching. Verify with `aivax_search_context` when the merge behavior is unclear.
- Public keys cannot use gateway slugs. Use the full UUID.

## Escalation

- A new gateway is needed: load `situations/design-gateway.md`.
- A skill or tool must be attached: load `situations/attach-skills-and-tools.md`.
- A parameter must be edited: load `situations/edit-gateway-safely.md`.
- A gateway is failing: load `references/observability/situations/diagnose-degradation.md`.
- A gateway is expensive: load `references/cost-monitoring/situations/optimize-spend.md`.

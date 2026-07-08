# AI Gateways

Use this sub-skill to work on AI Gateway runtimes: the AIVAX layer that combines model routing, instructions, RAG, skills, tools, MCP sources, protocol functions, multimodal preprocessing, and inference behavior behind an OpenAI-compatible endpoint.

This is for configuring and operating gateways in the user's AIVAX account. Do not inspect or modify AIVAX gateway source code, provider implementations, or server internals unless the user explicitly asks for source-code work.

## Situation Playbooks

- `situations/debug-failed-request.md`: gateway request failed, wrong tool, no answer, or model/provider error.
- `situations/inspect-routing.md`: identify configured and observed model/gateway/routing behavior.
- `situations/analyze-latency.md`: diagnose slow gateway or channel responses.
- `situations/analyze-cost-spike.md`: connect gateway spend to model, RAG, tools, sessions, or batch.

Use the listed API paths through `aivax_invoke_function` with hostless paths. Search AIVAX context before setting advanced or unfamiliar fields.

```text
aivax_invoke_function({
  "method": "GET",
  "path": "/api/v1/ai-gateways/<gateway-id>"
})

aivax_invoke_function({
  "method": "PATCH",
  "path": "/api/v1/ai-gateways/<gateway-id>",
  "body": {
    "parameters": {
      "modelName": "@openai/gpt-5-mini"
    }
  }
})
```

## Discovery Workflow

1. List candidate models before selecting or replacing models:

```text
aivax_list_models({ "name_filter": "gpt 5 mini" })
```

2. List gateways:

```text
GET /api/v1/ai-gateways
GET /api/v1/ai-gateways?filter=<name-or-model>
```

3. View the target gateway before patching:

```text
GET /api/v1/ai-gateways/<gateway-id>
```

4. Inspect related resources:

```text
GET /api/v1/skills
GET /api/v1/collections
GET /api/v1/web-chat-client
GET /api/v1/conversations?offsetminutes=120&filter=--gateway <gateway-id>
```

Do not patch a gateway from memory. Always read the current parameters first.

## Parameter Verification

Before setting advanced fields such as `sentinelOptions`, `builtinFunctionsOptions`, `enabledMultimodalFeatures`, `contextOverflowAction`, `mcpSources`, `protocolFunctions`, routing, or tool handler settings:

1. Inspect the current gateway shape.
2. Search AIVAX context for the field if behavior or payload format is unclear.
3. Verify model support with `aivax_list_models` when the field depends on model capability.
4. Patch only the field being changed.

## Gateway Creation

Use integrated models whenever possible:

```text
POST /api/v1/ai-gateways
{
  "name": "Support Agent",
  "parameters": {
    "baseAddress": "@integrated",
    "modelName": "@openai/gpt-5-mini",
    "systemInstruction": "You are a support assistant...",
    "knowledgeCollections": [],
    "skills": []
  }
}
```

For external OpenAI-compatible providers, use the provider base URL and preserve `apiKey` secrecy. Never print or summarize the key.

## Safe Editing

`PATCH /api/v1/ai-gateways/<id>` shallow-merges top-level gateway fields and top-level `parameters` fields. Send only fields that should change:

```text
PATCH /api/v1/ai-gateways/<gateway-id>
{
  "parameters": {
    "knowledgeBaseMaximumResults": 8,
    "knowledgeBaseMinimumScore": 0.3,
    "queryStrategy": "FullRewrite"
  }
}
```

Avoid sending copied full parameter objects unless the user asks for a full replacement and you have verified the current shape.

## Core Parameters

Model and provider:

- `baseAddress`: `@integrated` for AIVAX managed models, or an external OpenAI-compatible endpoint.
- `modelName`: model returned by `aivax_list_models` or a gateway/model slug accepted by the account.
- `apiKey`: external provider secret. Preserve or omit unless intentionally changing it.
- `temperature`, `topP`, `presencePenalty`, `frequencyPenalty`, `stop`, `maxCompletionTokens`, `reasoningEffort`, `verbosity`: only set when the selected model supports them.
- `contextMaximumSize` and `contextOverflowAction`: use to cap or handle long context. Actions include `Throw`, `Truncate`, `TruncateHard`, and `Compact`.

Instructions:

- `systemInstruction`: the base behavior of the gateway.
- `systemInstructionsSources`: external text resources appended to the system prompt.
- `userPromptTemplate`: transforms user messages before inference.
- `assistantPrefill` and `includePrefillingInMessages`: use only for compatible models.

RAG:

- `knowledgeCollections`: collection IDs queried before model call.
- `rerankerName`: commonly `lexical`; verify alternatives before use.
- `knowledgeBaseMaximumResults`: number of documents injected.
- `knowledgeBaseMinimumScore`: similarity cutoff from 0 to 1.
- `knowledgeUseReferences`: include referenced parent documents.
- `knowledgeUseMetaDescriptions`: include metadata descriptions in retrieved context.
- `queryStrategy`: `Plain`, `Concatenate`, `FullRewrite`, `UserRewrite`, or `QueryFunction`.
- `queryStrategyParameters`: `rewriteContextSize` and `concatenateContextSize`.

Tools and agent behavior:

- `skills`: flat list of AIVAX skill IDs. There is no native priority or primary skill concept.
- `hideToolsWithoutSkill`: hide tools not permitted by active skills.
- `alwaysVisibleTools`: tools that remain available even when hiding is enabled.
- `mcpSources`: external MCP servers exposed to the gateway.
- `protocolFunctions`: direct callback tool definitions.
- `protocolFunctionSources`: URLs returning protocol function definitions.
- `sentinelOptions.enabledFunctions`: built-in tools exposed through Sentinel reasoning.
- `builtinFunctionsOptions`: options for built-in web search, image generation, memory, and related tools.
- `enableBash` and `bashOptions`: controlled virtual Bash behavior.
- `toolContextCount`: how many previous tool results stay in context.
- `toolInvocationExplanations`: visible tool-call explanations when supported.
- `knownToolHandlerName`: use `native` or omit unless a compatibility handler is required.

Multimodal and moderation:

- `enabledMultimodalFeatures`: input media AIVAX may resolve and forward, such as images, audio, video, or files.
- `moderationParameters`: safety thresholds. Do not loosen moderation without a clear user requirement.

## Gateway Improvement Workflow

1. Capture current gateway details and recent conversations for evidence.
2. Identify the failure mode: poor answers, missing facts, tool failures, high latency, or high cost.
3. Apply one change at a time when possible.
4. Validate through a test conversation, conversation listing, RAG query, or account usage view.
5. Record the gateway ID, changed parameters, and validation evidence.

## Validation

After changes, verify the gateway view, linked chat clients, referenced skills/collections, at least one real conversation or inference smoke test, and absence of new errors or unexpected token growth.

Use a task-specific smoke test:

- Model swap: run one short representative prompt and compare model name, latency, token use, and answer quality.
- RAG attachment: ask a question that should retrieve from the new collection and inspect the conversation resources or RAG transaction.
- Tool or MCP source change: ask for the smallest action that should call the tool and verify the tool result is present.
- Cost or latency tuning: compare usage, token count, top K, output length, and recent conversations before and after the change.

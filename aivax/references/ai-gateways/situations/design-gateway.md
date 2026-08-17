# Design Gateway

Use when the agent must design a new AI Gateway from scratch — choosing the model, the instruction, the RAG collection(s), the skills, the tools, and the public-facing configuration.

## Objective

Produce a gateway definition that meets the user's goal with the right balance of capability, cost, latency, and quality.

## Preconditions

- The user goal is clear: what should the assistant do, who is the user, what is the entry point.
- The plan and balance are known.
- The relevant resources are accessible (collections, skills, MCP sources, provider keys).

## Decision Tree

1. What is the entry point? Web chat, messaging integration, API, or batch.
2. What is the model? Pick the cheapest model that meets the capability floor. See `references/text-inference/situations/choose-model.md`.
3. What is the system instruction? Short, specific, and named. Do not duplicate gateway skills.
4. What collections? One or more RAG collections. Each collection should be authoritative for a domain. See `references/rag/`.
5. What reranker? Default is `lexical`. Switch to a semantic reranker for higher precision. See `references/rerankers/`.
6. What skills? Skills are reusable instruction bundles attached by ID. Load `references/skill-development/situations/attach-skill-to-gateway.md` for the pattern.
7. What tools? MCP sources, protocol functions, built-in functions. Enable only what is needed. See `references/ai-gateways/situations/attach-skills-and-tools.md`.
8. What multimodal inputs? Set `enabledMultimodalFeatures` to the modalities the gateway should accept.
9. What moderation? Set `moderationParameters` to the safety threshold the user requires. Do not loosen moderation without a clear user requirement.
10. What context size? `contextMaximumSize` should be at least the model's context window. `contextOverflowAction` should be `Truncate` for most cases, `Compact` for long conversations, `TruncateHard` for safety-critical gateways.
11. What sampling? `temperature`, `topP`, `maxCompletionTokens`, `reasoningEffort`, `verbosity`. Set only what the chosen model supports.
12. What is the access pattern? Public keys (for browser or widget) or private keys (for server-side). Public keys strip tool surfaces.

## Construction

```text
POST /api/v1/ai-gateways
{
  "name": "<assistant-name>",
  "parameters": {
    "baseAddress": "@integrated",
    "modelName": "<chosen-model>",
    "systemInstruction": "<short-specific-instruction>",
    "knowledgeCollections": ["<collection-id>"],
    "rerankerName": "lexical",
    "knowledgeBaseMaximumResults": 8,
    "knowledgeBaseMinimumScore": 0.2,
    "skills": ["<skill-id-1>"],
    "contextMaximumSize": 128000,
    "contextOverflowAction": "Truncate",
    "maxCompletionTokens": 1024,
    "moderationParameters": { ... }
  }
}
```

## Validation

- The gateway view reflects the new configuration.
- A test conversation or inference smoke test produces the expected result.
- The linked resources (collections, skills, MCP sources) are valid.
- Recent conversations do not show new errors.
- The cost is within the user's cap.

## Limits

- Some parameters depend on the model. Verify with `aivax_list_models` or `aivax_search_context`.
- The system instruction is read on every call. Keep it short and stable for input caching.
- Skills are a flat list. There is no native priority. Order does not matter at the API level.

## Escalation

- A skill must be created: load `references/skill-development/situations/author-skill.md`.
- A collection must be created: load `references/rag/situations/design-rag-pipeline.md`.
- A model swap is needed: load `references/text-inference/situations/choose-model.md`.
- A chat client must be created: load `references/web-chat/`.

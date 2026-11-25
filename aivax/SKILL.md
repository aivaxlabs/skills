---
name: aivax
description: Operate AIVAX as a unified agent platform. Use when an agent needs to plan, execute, or validate work that touches AIVAX inference, background (async) generations, RAG, rerankers, agentic tests, voice (TTS, STT, realtime), image generation, text tools (classify, segment, describe, extract, summarize, translate, refine instructions), AI gateways, gateway workers, protocol functions, batch, web chat, account operations, cost monitoring, or observability. Routes to focused sub-skills instead of duplicating endpoint documentation. Covers the case when the AIVAX MCP is available and when only public docs are reachable.
---

# AIVAX

AIVAX is an AI orchestration platform: hosted or BYOK models, RAG collections, tools, skills, gateways, chat clients, batch, agentic tests, voice, image generation, and account APIs behind a single wallet and OpenAI-compatible inference surface. This skill does not memorize endpoints; it helps the agent pick the right capability, follow the right contract, and validate the result.

Do not use this skill to inspect, modify, build, or reason from AIVAX source code, repositories, deployment scripts, database models, or server internals unless the user explicitly asks for a separate source-code engineering task.

## Router Matrix

Start here. Load only the sub-skill the intent needs, and compose several only when the task truly spans capabilities. When two match, prefer the one closer to the user-facing outcome and the more specific one.

| Intent | Load |
| --- | --- |
| Choose a model, build chat-completion (or Responses / Anthropic Messages) requests, stream, attach tools, retry, or swap models for cost/latency/quality | `references/text-inference/` |
| Send images, audio, video, or files inside chat-completion messages; preprocess media to text | `references/multimodal/` |
| Generate images from text prompts | `references/image-generation/` |
| Synthesize speech from text (TTS) | `references/speech/` (TTS) |
| Transcribe audio to text (STT) | `references/speech/` (STT) |
| Open a realtime bidirectional voice session | `references/voice-realtime/` |
| Classify documents, segment text, describe media into text, extract schema fields, summarize, or translate documents | `references/text-tools/` |
| Review instructions, prompts, skills, or tool descriptions and get located findings with diff patches | `references/text-tools/` (instruction refinement) |
| Evaluate one text against fixed noul/choice/score criteria in a single call | `references/decisions/` |
| Search the web or extract content from URLs/files via direct API calls | `references/web/` |
| Build a RAG pipeline: collection, ingestion, query, answer, grounding | `references/rag/` |
| Rerank candidate documents; choose Reflex, Cohere, Qwen, Jina, NVIDIA, or lexical reranker | `references/rerankers/` |
| Create, run, evaluate, or compare agentic tests | `references/agentic-tests/` |
| Create, list, rotate, or delete API keys; inspect account, plan, subscription, invoices, privacy settings (zero data retention, conversation and RAG logging), console two-factor sign-in, or salt | `references/account/` |
| Investigate a cost spike, balance drop, or spend optimization | `references/cost-monitoring/` |
| Correlate traces, diagnose latency/error, audit a conversation | `references/observability/` |
| Create, edit, or operate an AI gateway (model + instructions + RAG + tools + skills) | `references/ai-gateways/` |
| Implement or debug a gateway worker (message.received, tool.called, tool.fetched, message.completed) | `references/gateway-workers/` |
| Define or serve a protocol function (server-side tool with HTTP callback) | `references/protocol-functions/` |
| Author or attach an account-scoped skill under `/api/v1/skills` | `references/skill-development/` |
| Operate a web chat client, session, or messaging integration | `references/web-chat/` |
| Inspect or cancel scheduled hook activations for a web chat client | `references/hook-schedules/` |
| Read or mutate files inside the shell sandbox (account + X-Shell-User-Id) | `references/shell-io/` |
| Save, search, or manage persisted memories in the `@memories` collection | `references/memories/` |
| Check account notifications or the unread count | `references/notifications/` |
| Design or debug a batch workflow/job/items | `references/batch/` |
| Run a call in the background with `async`, or read, cancel, or delete queued generations | `references/resilience/` (async generation) |
| Combine multiple AIVAX capabilities into a single workflow | `references/composition/` |
| Handle rate limits, transient errors, retries, idempotency, fallback | `references/resilience/` |
| Anything transversal: which tool to call, how to mutate safely, how to interpret errors | `references/platform-rules/` |

"The agent's last answer was wrong" usually means `references/rag/` (grounding) or `references/text-inference/` (prompt/model); escalate to `references/observability/` only with telemetry evidence of a failure.

## Discovery And No-MCP Mode

Check whether the AIVAX Account Management MCP is available before relying on a tool.

- With the MCP, prefer `aivax_invoke_function` (account-scoped API call with a hostless path), `aivax_list_models`, and `aivax_search_context` over direct HTTP. A sub-skill that says "verify the field name with the docs" always means an `aivax_search_context` call, not a guess.
- Without it, use public surfaces and document every call before making it: <https://inference.aivax.net/apidocs/llms.txt> and <https://docs.aivax.net/docs/overview>. Send `X-Response-Truncating: agent-optimized` on HTTP calls. Ask the user for the base URL and API key surface when the integration is not configured; do not infer hosts, credentials, or internal paths.

Either way, read the live source of truth before mutating and never invent field names. Tool details are in `references/platform-rules/`.

## Default Workflow

1. Classify the intent and load the right sub-skill from the Router Matrix.
2. Discover the environment: MCP available? Auth configured? Plan and balance sufficient? Model or gateway exists?
3. Read current state for every resource you intend to touch.
4. If the contract needs a capability the loaded sub-skill lacks, switch to the right one; do not invent endpoints.
5. Apply the smallest change that satisfies the request. Prefer reversible operations.
6. Validate through the same user-facing or telemetry path: a gateway change by a test conversation, RAG by a query, a chat client by a session or talk URL, batch by item inspection or export.
7. Report what changed, what was verified, resource and request IDs, balance snapshot and cost impact, and any remaining risk.

## Global Safety Rules

These apply in addition to the rules inside each sub-skill.

- Preserve secrets. Never print API keys, provider keys, integration tokens, webhook secrets, salts, chat access keys, or private credentials.
- Treat destructive or costly operations as explicit-only: delete, reset, clear, roll salt, imports that overwrite many resources, sync imports that remove missing records, bulk cancellation, model swaps in production, and rate-limit changes.
- Export or capture current configuration before large imports, destructive edits, broad migrations, skill imports, collection resets, or gateway rewrites.
- For shallow-merge endpoints, send only the fields that should change.
- Keep AIVAX `systemInstruction` separate from account skills. Skills are a flat list attached to gateways; do not assume native priority or a primary skill.
- For integrated models, prefer `baseAddress: "@integrated"` and a model returned by `aivax_list_models`. For external providers, preserve existing `apiKey` values when patching unrelated gateway parameters.
- Use the account API (`aivax_invoke_function`) for resource work. The Account Management MCP is the operating interface; local source code is not.
- Do not include media payloads, full transcripts, hidden reasoning, or private user data in final responses unless the user explicitly needs that exact data.
- Keep prices, numeric plan limits, plan/account multipliers, minimum balances, and negotiated terms out of reusable skills. Consult [Pricing](https://docs.aivax.net/docs/pricing) and [Plans and limits](https://docs.aivax.net/docs/limits); route account-specific terms to the authorized administrator.

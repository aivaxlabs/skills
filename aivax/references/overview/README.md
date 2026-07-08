# Product Overview

Use this reference when an agent needs to understand AIVAX before operating a user account. It explains the platform, the product families, how resources fit together, and how to choose the right operating surface.

This reference is for product and account orientation. Do not inspect or modify AIVAX source code, repositories, deployment scripts, database models, or server internals unless the user explicitly asks for a separate source-code engineering task.

## Contents

- What AIVAX Is
- Product Families
- How The Pieces Fit Together
- Core Resource Concepts
- When To Use Each Product Area
- Common User Journeys
- Product Decision Rules
- Operating Principles

## What AIVAX Is

AIVAX is an AI orchestration platform for production AI operations. It unifies model inference, RAG, agentic tools, moderation, structured output handling, batch processing, chat delivery channels, observability, and cost management behind account-level resources and OpenAI-compatible API patterns.

The central idea is a single operating layer for agents that need knowledge, actions, governance, and delivery channels without manually stitching together separate model providers, embedding pipelines, tool servers, moderation systems, logs, and chat clients.

In user-facing terms, AIVAX helps teams:

- Create AI agents through configurable gateways.
- Connect those agents to knowledge collections.
- Expose tools, MCP servers, and protocol functions to the model.
- Publish the same agent through API, web widget, Telegram, WhatsApp, or other chat clients.
- Run batch processing and evaluations over many items.
- Monitor conversations, model usage, token spend, resource cost, quality, and errors.

## Product Families

| Product area | What it is | What it is used for | Main account resources |
| --- | --- | --- | --- |
| Model Gateway | A unified model access layer with integrated models and optional external providers. | Use GPT, Claude, Gemini, Llama, and other models through one account balance or through BYOK provider keys. | Models, integrated model metadata, provider settings, balance, usage. |
| AI Gateways | The orchestration runtime for an agent or inference pipeline. | Configure model choice, instructions, RAG, tools, MCP sources, moderation, structured output, multimodal preprocessing, and OpenAI-compatible inference behavior. | AI gateway, gateway parameters, skills, collections, tools, MCP sources, protocol functions. |
| RAG and Semantic Search | Managed knowledge collections and retrieval pipelines. | Index documents, retrieve relevant passages, enrich prompts, diagnose retrieval quality, and provide grounded answers. | Collections, documents, vectors, query results, RAG transactions, performance reports. |
| Chat Clients | User-facing delivery surfaces connected to a gateway. | Publish an agent as a web widget, web chat session, Telegram bot, WhatsApp integration, or other messaging channel. | Web chat clients, sessions, access keys, talk URLs, integrations, client parameters. |
| Batch Jobs | Asynchronous AI processing at scale. | Run the same instruction, model/gateway, schema, tools, and validation over many records, files, JSONL lines, tickets, URLs, or independent items. | Batch workflows, jobs, items, exports, validation results, confidence, per-item costs. |
| Conversation Logs | Stored conversation traces and diagnostics. | Understand what happened in a request, debug tools/RAG/instructions, audit customer interactions, sample quality, and connect issues to resources. | Conversations, messages, tools, usage objects, resources, exports, error states. |
| Account Skills | Runtime instruction packages attached to gateways. | Add reusable task-specific behavior and allowed tool scopes without duplicating gateway system instructions. | Skills, instruction sources, allowed tool names, gateway `parameters.skills`. |
| Observability and Cost Monitoring | Usage, balance, model pricing, resource attribution, latency, and cost analysis. | Explain spend, identify high-cost resources, detect runaway jobs or tool loops, and recommend safe optimizations. | Balance, usage details, model pricing, resource discriminations, subscription reserve, storage. |
| Account Management MCP | The agent operating interface for account work. | Let agents inspect and mutate AIVAX account resources through scoped tools instead of source code or manual console work. | `aivax_invoke_function`, `aivax_list_models`, `aivax_search_context`. |

## How The Pieces Fit Together

A common AIVAX production flow is:

1. A user request enters through an API call, web chat client, or messaging integration.
2. The request is routed to an AI Gateway.
3. The gateway applies model settings, system instructions, account skills, moderation, structured output rules, and context management.
4. If configured, the gateway queries RAG collections and injects relevant passages.
5. If configured, the model can use tools, protocol functions, MCP sources, built-in functions, or a shell sandbox.
6. AIVAX records conversation, usage, resources, cost, errors, and telemetry.
7. The response returns through the original channel.

This means most account work starts by identifying the affected resource chain:

```text
Chat Client or API Key
  -> AI Gateway
  -> Model + Instructions + Skills
  -> RAG Collections
  -> Tools / MCP Sources / Protocol Functions
  -> Conversation Logs + Usage
```

Batch work follows a related but separate chain:

```text
Batch Workflow
  -> Model or AI Gateway
  -> Result Schema + Validation Instruction + Tools
  -> Batch Job
  -> Batch Items
  -> Exports + Usage
```

## Core Resource Concepts

- Account: the billing, authorization, limits, resources, and configuration boundary.
- Authorization key or API key: a credential/resource that allows requests into account APIs or gateway endpoints. Treat it as sensitive.
- Integrated model: a model available through AIVAX managed access and account balance. Use `aivax_list_models` before selecting one.
- External provider model: a model accessed through an external OpenAI-compatible base URL and provider API key. Preserve secrets when patching.
- AI Gateway: the main runtime object for inference and agents. It is the place where model, instruction, RAG, tools, skills, and moderation meet.
- Gateway parameters: the configuration object inside a gateway. Many updates are shallow merges, so patch only intended fields.
- Collection: a RAG knowledge base containing documents and vector/search metadata.
- Document: an indexed knowledge item inside a collection, often with reference, tags, metadata, and contents.
- RAG transaction: telemetry for an actual retrieval operation, useful for diagnosing relevance, scores, process time, and query behavior.
- Chat client: a configured user-facing client connected to a gateway.
- Session: a chat-client conversation access unit with tag, expiry, context, metadata, access key, and talk URL.
- Conversation: a stored record of messages, model, tool calls, resources, usage, errors, and timestamps.
- Skill: an account-stored instruction package, attached to gateways by ID. It is not a local Codex skill folder.
- Batch workflow: reusable batch definition with instructions, schema, model/gateway, tools, validation, retry policy, and handling rules.
- Batch job: one execution run under a workflow.
- Batch item: one input/output unit processed by a job.
- Usage resource: cost attribution that links spend back to models, gateways, collections, chat clients, batch jobs, API keys, or other resources.

## When To Use Each Product Area

Use AI Gateways when the user asks to:

- Create or tune an agent.
- Change model, temperature, context window, routing, moderation, or structured output behavior.
- Attach RAG, skills, tools, MCP sources, or protocol functions.
- Expose an OpenAI-compatible endpoint with AIVAX orchestration.

Use Model Gateway and model discovery when the user asks to:

- Choose a model.
- Compare price, speed, context window, capability, provider, or plan availability.
- Move from one provider/model to another.
- Decide between integrated model access and BYOK external provider access.

Use RAG and Semantic Search when the user asks to:

- Add or diagnose knowledge.
- Improve grounded answers.
- Analyze retrieval relevance, low scores, stale vectors, missing documents, or noisy context.
- Run semantic queries or quality reports.

Use Chat Clients when the user asks to:

- Publish an agent to a website or messaging channel.
- Create a session or talk URL.
- Configure iframe origins, UI text, colors, uploads, audio, rate limits, or Telegram/WhatsApp integrations.
- Diagnose a channel-specific issue.

Use Batch Jobs when the user asks to:

- Process many records with the same AI task.
- Extract, classify, enrich, validate, summarize, or evaluate data at scale.
- Import JSONL/text/files, monitor progress, retry failures, or export results.

Use Conversation Analysis when the user asks to:

- Debug a bad answer.
- Understand a tool call, RAG failure, model behavior, latency, or error.
- Audit conversations in bulk.
- Connect an observed issue to a gateway, collection, skill, tool, chat client, session, or model.

Use Skill Development when the user asks to:

- Create account skills for gateway behavior.
- Refine instructions reusable across gateways.
- Restrict or expose tool access by skill.
- Import/export account skills.

Use Cost Monitoring when the user asks to:

- Explain spend, balance, usage, model costs, storage, or subscription reserve.
- Find high-cost resources.
- Reduce token usage, batch spend, RAG overhead, tool loops, or expensive model usage.

Use Account Management when the user asks to:

- Inspect the account broadly.
- Find resource IDs.
- Plan a multi-resource change.
- Safely apply changes through the MCP/API.

## Common User Journeys

Create a grounded support agent:

1. Create or choose a RAG collection.
2. Add/import documents and verify retrieval.
3. Create an AI Gateway with a suitable model and instructions.
4. Attach the collection and any account skills/tools.
5. Create a chat client or API integration.
6. Run a test conversation and inspect the conversation record.
7. Check usage and tune cost/quality.

Improve a poor answer:

1. View the conversation.
2. Identify whether the issue came from instructions, model choice, RAG, tools, session context, chat client, or cost/latency constraints.
3. Inspect the linked gateway and related resources.
4. Apply the smallest change.
5. Re-test and inspect the new conversation.

Run a batch enrichment:

1. Define input unit and output schema.
2. Choose model or gateway.
3. Create or reuse a workflow.
4. Import a small pilot job.
5. Review outputs, validation, errors, confidence, and cost.
6. Scale the job only after the pilot is acceptable.
7. Export results and reconcile by stable input IDs.

Reduce spend:

1. Compare balance and usage over 24h, 7d, and 30d.
2. Rank spend by model, category, and resource.
3. Inspect high-cost gateways, collections, conversations, or batch jobs.
4. Recommend changes by savings and risk.
5. Get approval before changes that affect quality, data, active jobs, or customer-facing behavior.

## Product Decision Rules

- If the user mentions an agent, assistant, API endpoint, model behavior, tools, instructions, RAG attachment, or OpenAI-compatible endpoint, start with AI Gateways.
- If the user mentions documents, knowledge, search, citations, retrieval, vectors, or hallucination from missing facts, start with RAG.
- If the user mentions website chat, iframe, talk URL, Telegram, WhatsApp, sessions, user tags, or channel behavior, start with Chat Clients.
- If the user mentions many rows, CSV, JSONL, files, queue, retries, export, validation, or confidence, start with Batch Jobs.
- If the user mentions an exact bad answer, tool loop, failed request, latency, or audit, start with Conversation Analysis.
- If the user mentions reusable instructions, allowed tools, skill import/export, or gateway-attached instructions, start with Skill Development.
- If the user mentions balance, usage, cost, pricing, storage, tokens, subscription models, or runaway spend, start with Cost Monitoring.
- If the user is unsure, start with Account Management and list resources before changing anything.

## Operating Principles

- Treat AIVAX documentation, API responses, and MCP tools as the source of truth for account operations.
- Keep product explanation user-centered: explain what the user can do and how to verify it, not how the platform is implemented.
- Do not overfit one product area. Many issues span gateway, RAG, chat client, conversation, and usage resources.
- Prefer reading the resource graph before mutating any resource.
- Keep account skills distinct from local agent skills. AIVAX account skills are runtime resources under `/api/v1/skills`.
- Treat public delivery artifacts such as talk URLs, access keys, external user tags, and session context as sensitive.
- Keep cost and quality together. A cheaper model, lower RAG top K, or stricter tool access is only an improvement if the user's required behavior still works.

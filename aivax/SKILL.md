---
name: aivax
description: Operate AIVAX user accounts through the AIVAX Account Management MCP and account APIs, not by changing AIVAX source code. Use when an agent such as Codex, Claude, or Antigravity needs to understand AIVAX, choose the right AIVAX product surface, inspect resources, debug failures, analyze RAG or conversations, configure AI gateways or chat clients, run batch jobs, manage account skills, monitor costs, or safely mutate account state. This skill is an operational router to MCP tools, safe mutations, gotchas, and task-specific situation playbooks.
---

# AIVAX

Use this skill as an operating manual for agents working with an authenticated AIVAX account. The goal is not to describe AIVAX passively; the goal is to help the agent choose the right account resource, call the right MCP tool in the right order, avoid common mistakes, and validate the result.

## Scope Boundary

Use AIVAX as a platform through the Account Management MCP, account APIs, documentation, and live account resources.

Do not inspect, modify, build, deploy, or reason from the AIVAX product source code unless the user explicitly asks for a separate source-code engineering task. When this skill is active, treat MCP tools and public/account API surfaces as the operating interface.

## Core Behavior

1. Identify the AIVAX area involved:
   - AI Gateways
   - RAG and semantic search
   - Batch Jobs
   - Cost Monitoring
   - Account Management MCP
   - Conversation Analysis
   - Chat Clients
   - Account Skills
   - Operations and debugging
2. Before taking action:
   - Read the relevant area README.
   - Read the relevant situation playbook when one exists.
   - Check `references/mcp/tool-selection.md` if a tool call is required.
   - Prefer read-only inspection before mutation.
3. For mutations:
   - Explain the intended change when risk is non-trivial.
   - Read current state first.
   - Identify the exact target resource ID.
   - Apply the smallest safe change.
   - Verify the result after the change.
4. For debugging:
   - Inspect the failing state or reproduce with the smallest safe request.
   - Collect conversation/resource/usage evidence.
   - Identify the likely layer: gateway, RAG, tool, model, chat client, batch, account, or cost.
   - Recommend or apply the smallest fix.
   - Verify with the same path that failed.

## MCP Contract

Prefer the configured AIVAX Account Management MCP whenever it is available.

Core tools:

- `aivax_invoke_function`: invoke account-scoped AIVAX API functions. Use hostless paths such as `/api/v1/ai-gateways`; never include a host.
- `aivax_list_models`: list integrated models, pricing, providers, context windows, capabilities, plan availability, and flags.
- `aivax_search_context`: search AIVAX documentation and API references before using unfamiliar fields or endpoints.

Read first:

- `references/mcp/tools.md`
- `references/mcp/tool-selection.md`
- `references/mcp/safe-mutations.md`
- `references/mcp/errors.md`

## Reference Router

Use only the references needed for the task.

Product orientation:

- `references/overview/README.md`: what AIVAX is, product families, resource relationships, common journeys, and decision rules.

MCP and safety:

- `references/mcp/README.md`: base account operations and common account surfaces.
- `references/mcp/tools.md`: MCP tool contracts and call shapes.
- `references/mcp/tool-selection.md`: which MCP/API call to use for common intents.
- `references/mcp/safe-mutations.md`: mutation rules, confirmations, and validation.
- `references/mcp/errors.md`: common API/MCP error handling patterns.

AI Gateways:

- `references/ai-gateways/README.md`: gateway parameters, creation, editing, and validation.
- `references/ai-gateways/situations/debug-failed-request.md`
- `references/ai-gateways/situations/inspect-routing.md`
- `references/ai-gateways/situations/analyze-latency.md`
- `references/ai-gateways/situations/analyze-cost-spike.md`

RAG:

- `references/rag/README.md`: collections, query, transactions, performance reports, and remediation.
- `references/rag/situations/diagnose-bad-answer.md`
- `references/rag/situations/inspect-retrieval.md`
- `references/rag/situations/debug-missing-documents.md`
- `references/rag/situations/evaluate-quality.md`

Batch Jobs:

- `references/batch-jobs/README.md`: workflows, jobs, items, imports, monitoring, retry, export.
- `references/batch-jobs/situations/debug-failed-job.md`

Cost Monitoring:

- `references/cost-monitoring/README.md`: balance, usage, model spend, resource spend, storage, reserve, optimization.
- `references/cost-monitoring/situations/investigate-cost-spike.md`

Conversations and Chat Clients:

- `references/conversations/conversation-analysis.md`: stored conversation diagnosis and exports.
- `references/conversations/chat-clients.md`: web chat clients, sessions, origins, integrations, and channel validation.

Operations:

- `references/operations/debugging.md`: cross-resource debugging workflow.

Account Skills:

- `references/skill-development/README.md`: account skills under `/api/v1/skills`, imports, exports, allowed tools, gateway attachment.

## Default Workflow

1. Determine whether the user wants orientation, inspection, debugging, creation, mutation, or optimization.
2. Load `references/overview/README.md` if the task is broad or the product area is unclear.
3. Load the area README and any matching situation playbook.
4. Use `references/mcp/tool-selection.md` to choose calls.
5. Read current state through `aivax_invoke_function`.
6. Search docs with `aivax_search_context` when schema, behavior, or enum values are unclear.
7. For model changes, call `aivax_list_models` before selecting a model.
8. For mutations, follow `references/mcp/safe-mutations.md`.
9. Validate through the same user-facing path or telemetry path that matters for the task.
10. Report changed resources, validation evidence, and remaining risk.

## Global Safety Rules

- Preserve secrets. Never print API keys, provider keys, integration tokens, webhook secrets, salts, chat access keys, or private credentials.
- Do not use local AIVAX repository files, implementation details, database code, or server internals as the operating path for account tasks.
- Treat destructive operations as explicit-only actions: delete, reset, clear, roll salt, imports that overwrite many resources, sync imports that remove missing records, and bulk cancellation.
- Export or capture current configuration before large imports, destructive edits, broad migrations, skill imports, collection resets, or gateway rewrites.
- For shallow-merge endpoints, send only the fields that should change.
- Keep AIVAX `systemInstruction` separate from account skills. Skills are a flat gateway-attached list; do not assume native priority, weights, or a primary skill.
- For integrated models, prefer `baseAddress: "@integrated"` and a model name returned by `aivax_list_models`.
- For external providers, preserve existing `apiKey` values when patching unrelated gateway parameters.

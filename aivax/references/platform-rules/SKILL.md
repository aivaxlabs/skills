---
name: aivax-platform-rules
description: Cross-cutting rules for AIVAX work — which tool to call, how to mutate safely, how to interpret errors, how to discover capabilities when the MCP is not available. Load when the agent is about to make an account-scoped call, mutate a resource, or read an error message that does not match any single sub-skill.
---

# Platform Rules

This sub-skill is transversal. It does not own a specific capability; it owns the rules every other AIVAX sub-skill relies on. If a sub-skill conflicts with this one, this one wins for cross-cutting rules; the sub-skill wins for its specific contract.

## Operating Files

- `tool-selection.md`: how to map an intent to a tool call (MCP or HTTP).
- `safe-mutations.md`: required sequence for any account mutation, plus approval gates.
- `error-handling.md`: how to interpret status codes, error classes, and rate limits.

## Scope Boundary

Operate AIVAX through the user's account surface (MCP tools, account APIs, public docs, and live resources). Do not use local AIVAX source code, repositories, deployment scripts, database models, or server internals to decide what to do on an account unless the user explicitly asks for source-code work.

## Core MCP Contract

When the AIVAX Account Management MCP is configured, the three core tools are:

- `aivax_invoke_function` — invoke any account-scoped AIVAX API with a hostless path such as `/api/v1/ai-gateways` and an optional JSON `body`. The server forwards the current authorization.
- `aivax_list_models` — list integrated models and AI gateways with capabilities, pricing, context window, modality, plan availability, and flags. Use `name_filter` for focused discovery.
- `aivax_search_context` — search AIVAX documentation and API reference before acting on unfamiliar fields, endpoints, enums, or behaviors. Use `search_type: "api-function-reference"`, `"documentation-manual"`, or `"all"`.

If a tool is unavailable in the current client, use only an equivalent authenticated MCP or account API surface exposed by that client. Ask the user for the base URL and authorization mechanism when neither is configured.

## Fallback HTTP Contract

When the MCP is not available, use direct HTTPS calls to `https://inference.aivax.net`. Authenticate with one of:

- `Authorization: Bearer <API_KEY>` (preferred for server-to-server).
- `Authorization: Basic <base64(username:api_key)>`.
- `?api-key=<API_KEY>` (only when the client cannot send headers).

For LLM-friendly responses, send `X-Response-Truncating: agent-optimized` on every call. Keep payloads compact and avoid media unless required.

## Decision Order

For any task that touches AIVAX, follow this order:

1. Classify the intent and load the right sub-skill (see `SKILL.md`).
2. Discover the environment: auth, plan, balance, MCP availability, model or gateway availability.
3. Read current state before any mutation.
4. If the contract is unclear, search AIVAX context (MCP) or `docs.aivax.net` / `inference.aivax.net/apidocs/llms.txt` (fallback).
5. Apply the smallest change.
6. Validate through the same path the user will use.
7. Report changed resources, IDs, evidence, and remaining risk.

## Global Safety Rules

- Preserve secrets. Never print API keys, provider keys, integration tokens, webhook secrets, salts, chat access keys, or private credentials.
- Treat destructive operations as explicit-only: delete, reset, clear, roll salt, sync imports that remove records, bulk cancellation.
- Export or capture current configuration before large imports, destructive edits, broad migrations, skill imports, collection resets, or gateway rewrites.
- For shallow-merge endpoints, send only the fields that must change.
- Keep AIVAX `systemInstruction` separate from account skills. Skills are a flat list attached to gateways; do not assume native priority or a primary skill.
- For integrated models, prefer `baseAddress: "@integrated"` and a model returned by `aivax_list_models`.
- For external providers, preserve existing `apiKey` values when patching unrelated gateway parameters.

## Validation Checklist

A change is only "done" when all relevant items below are verified.

- The changed resource can be viewed by ID after the update.
- Related listings reflect the change.
- Gateway changes have a functional smoke test or recent conversation check.
- RAG changes have a retrieval query or transaction or quality-report check.
- Chat-client changes have a session, talk URL, or integration check.
- Batch changes have workflow, job, item, and export or item-view verification.
- Cost-sensitive changes include a before/after or expected cost impact.
- Secrets are not exposed in any output, log, or final response.

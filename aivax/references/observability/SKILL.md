---
name: aivax-observability
description: Correlate AIVAX requests, conversations, and traces; diagnose latency, errors, and degradation; audit stored conversations. Load when the user reports a production failure, when the agent must trace a request across stages, or when the agent must audit conversation records for compliance or quality review.
---

# Observability

This sub-skill owns the operational visibility into AIVAX work. The discipline is to anchor analysis in conversation records, messages, usage objects, resources, timestamps, error messages, tools, response schemas, and related gateway/chat-client configuration — not in guesses.

## Operating Files

- `situations/correlate-request-trace.md`: follow a request from the entry point through every stage and resource.
- `situations/diagnose-degradation.md`: identify why a gateway, chat client, or batch job is slow or erroring.
- `situations/audit-conversation.md`: review stored conversations for compliance, quality, or post-mortem analysis.

## When To Use Observability

Use this sub-skill when the agent must:

- Trace a specific request from the entry point (API, web chat, integration) to the final response.
- Diagnose latency or error rate that has increased on a gateway, chat client, or batch job.
- Audit stored conversations for compliance, quality, or customer support.
- Correlate a request ID with a conversation, a transaction, a job item, or a hook schedule.
- Confirm that a recent change had the intended effect (or did not).

Do not use this sub-skill for:

- Inspecting a single resource in isolation (use the relevant capability sub-skill).
- A one-off question about a specific resource (use the relevant capability sub-skill).

## Operating Surfaces

- `GET /api/v1/conversations`: list conversations.
- `GET /api/v1/conversations/<id>`: view a conversation.
- `DELETE /api/v1/conversations/<id>`: delete a conversation (destructive).
- `GET /api/v1/conversations/management/export.jsonl?period=<window>&media=<mode>&thinking=<mode>&truncate=<n>`: bulk export.
- `GET /api/v1/conversations/<id>/export.json`: single conversation export.
- `GET /api/v1/information/usage`: usage for a period.
- `GET /api/v1/information/balance`: current health.

## Listing Filters

The conversations listing accepts filters such as `--gateway <id>`, `--chat-client <id>`, `--chat-session <id>`, `--model <name>`, `--user <external-user-id>`, `--api-key <id>`. URL-encode values that include spaces or flags.

## Bulk Export Options

- `period`: `2h`, `1d`, `7d`, `30d`.
- `media`: `text` or `include`.
- `thinking`: `visible`, `all`, `none`.
- `truncate`: maximum token estimate per conversation; `0` disables truncation.

Use `media: text` and `thinking: visible` for default exports. Use `media: include` and `thinking: all` only when the user needs the full data; the export will be large and may include sensitive information.

## Hygiene

- Do not paste full transcripts, media contents, hidden reasoning, API key identifiers, external-user identifiers, access keys, secrets, or private personal data in final responses unless the user explicitly needs that exact data.
- Prefer short evidence excerpts, redacted IDs, counts, and representative message summaries.
- When exporting, choose the smallest `period` and the least inclusive `media` and `thinking` modes that satisfy the task.

## Validation

- The trace or the conversation is identified.
- The relevant fields are extracted (timestamps, resources, error messages, tool calls, usage).
- The cause or the audit conclusion is supported by evidence.
- The trace ID is preserved.
- The report is reproducible: same conversation IDs, same filters, same result.

## Limits

- The conversations listing includes previews, not full content. The view endpoint may shorten very long message lists. Use export for full audits.
- Usage data can lag real-time spend. Do not promise immediate cost deltas.
- Deletion is destructive. Use only when the user explicitly asks.

## Escalation

- The trace points to a specific gateway: load `references/ai-gateways/`.
- The trace points to a RAG transaction: load `references/rag/situations/debug-bad-answer.md`.
- The trace points to a batch item: load `references/batch/situations/debug-failed-job.md`.
- The trace points to a chat client or session: load `references/web-chat/`.
- The cost is unexpectedly high: load `references/cost-monitoring/situations/investigate-cost-spike.md`.

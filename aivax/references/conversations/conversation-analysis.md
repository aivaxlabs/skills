# Conversation Analysis

Use this sub-skill to diagnose what actually happened in stored AIVAX conversations. Anchor analysis in conversation records, messages, usage objects, resources, timestamps, error messages, tools, response schemas, and related gateway/client configuration.

This is for analyzing account conversation records and related AIVAX resources. Do not inspect AIVAX storage, inference, integration, or UI source code unless the user explicitly asks for a separate source-code investigation.

Use the listed API paths through `aivax_invoke_function`. Search AIVAX context before using unfamiliar filters, export options, or conversation fields.

```text
aivax_invoke_function({
  "method": "GET",
  "path": "/api/v1/conversations?offsetminutes=120"
})

aivax_invoke_function({
  "method": "GET",
  "path": "/api/v1/conversations/<conversation-id>"
})
```

## List Conversations

Start broad, then filter:

```text
GET /api/v1/conversations?offsetminutes=120
GET /api/v1/conversations?offsetminutes=1440&filter=--gateway <gateway-id>
GET /api/v1/conversations?offsetminutes=1440&filter=--chat-client <client-id>
GET /api/v1/conversations?offsetminutes=1440&filter=--chat-session <session-id>
GET /api/v1/conversations?offsetminutes=1440&filter=--model <model-name>
GET /api/v1/conversations?offsetminutes=1440&filter=--user <external-user-id>
GET /api/v1/conversations?offsetminutes=1440&filter=--api-key <api-key-id>
```

The listing includes previews, model, token count, external user, error state, and usage resources. Use it to find candidates, not to make final conclusions.

## View One Conversation

```text
GET /api/v1/conversations/<conversation-id>
```

Inspect:

- `origin`: direct API, web chat, integration, or other source.
- `modelName`: actual model used.
- `requestId`: useful for linking usage and logs.
- `tools`: tool definitions made available.
- `usageObject`: prompt, completion, and total tokens when present.
- `resources`: gateway, chat client, session, collection, API key, batch item, or other resource attribution.
- `timestamps`: created/updated timing.
- `errorMessage`: provider, tool, validation, or platform errors.
- `messages`: user, assistant, system, tool, and reasoning messages.
- `metadata`: channel or integration context.

The view endpoint may shorten very long message lists. Use export for full audits.

## Export Conversations

Bulk export:

```text
GET /api/v1/conversations/management/export.jsonl?period=7d&media=text&thinking=visible&truncate=0
```

Options:

- `period`: `2h`, `1d`, `7d`, or `30d`.
- `media`: `text` or `include`.
- `thinking`: `visible`, `all`, or `none`.
- `truncate`: maximum token estimate per conversation; `0` disables truncation.

Single conversation export:

```text
GET /api/v1/conversations/<conversation-id>/export.json
```

Use JSONL exports for batch audits, metric sampling, regression analysis, or offline review. Keep media and reasoning exports minimal unless needed because they can be large and sensitive.

Do not paste full transcripts, media contents, hidden reasoning, API key identifiers, external-user identifiers, access keys, secrets, or private personal data in final responses unless the user explicitly needs that exact data. Prefer short evidence excerpts, redacted IDs, counts, and representative message summaries.

## Individual Diagnosis Workflow

1. View the conversation.
2. Identify the user goal and expected outcome.
3. Map every tool call to its result. Missing or mismatched tool results are high-signal.
4. Check whether RAG context was relevant, absent, repeated, or stale.
5. Compare prompt growth and token counts across turns.
6. Check model, gateway, skill, and client resources.
7. Inspect the linked gateway and chat client if the issue looks systemic.
8. Classify the root cause and propose the smallest fix.

Do not conclude from listing previews alone. Assign confidence based on evidence:

- High: full conversation view or export plus linked resource details support the cause.
- Medium: conversation view supports the cause, but linked resource details or validation are missing.
- Low: only listings, previews, or inferred patterns are available.

Common root cause categories:

- Retrieval gap: wrong collection, poor query strategy, missing content, stale vectors, or low score.
- Instruction conflict: gateway system instruction and skills disagree or repeat.
- Tool exposure issue: tool hidden by skill settings, MCP source failing, wrong protocol function, or handler mismatch.
- Tool loop: model repeatedly calls a tool despite successful results.
- Context issue: session `extraContext`, integration metadata, or old messages dominate the prompt.
- Client/session issue: wrong chat client, expired/refreshed session, wrong external user tag, or channel debounce.
- Model mismatch: selected model lacks needed capability, context window, tool support, or multimodal behavior.
- Cost/latency issue: excessive context, high top K RAG, many tool calls, long output, or expensive model.

## Bulk Analysis Workflow

1. Export the target period.
2. Segment by gateway, chat client, session, external user, model, and error state.
3. Sample both successful and failed conversations.
4. Count recurring issues: error messages, repeated tool names, high token counts, missing RAG evidence, and unanswered user intents.
5. Connect each pattern to a resource that can be fixed.
6. Use the focused sub-skill for the actual remediation.

## Deletion

Conversation deletion is destructive and should be rare:

```text
DELETE /api/v1/conversations/<conversation-id>
```

Use only when the user explicitly asks to remove a stored conversation or when a privacy cleanup request clearly covers it.

## Report Format

For one conversation, report conversation ID, model, origin, updated time, relevant resources, observed failure, evidence, root cause, confidence, fix path, and validation. For bulk analysis, report export window, filters, reviewed count, patterns, representative IDs, and recommended changes by resource.

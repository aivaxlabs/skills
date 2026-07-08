# Tool Selection

Use this file to map user intent to MCP tools and account API paths.

## General Rules

- Use `aivax_invoke_function` for account resources.
- Use `aivax_list_models` for model selection, model replacement, pricing, capability, context, and provider questions.
- Use `aivax_search_context` before unknown endpoint shapes, enum values, multipart forms, import/export behavior, or advanced gateway fields.

## Gateway Intents

Inspect gateways:

1. `GET /api/v1/ai-gateways`
2. `GET /api/v1/ai-gateways/<gateway-id>`

Change gateway model or routing:

1. `aivax_list_models({ "name_filter": "<candidate>" })`
2. `GET /api/v1/ai-gateways/<gateway-id>`
3. `PATCH /api/v1/ai-gateways/<gateway-id>` with only the changed `parameters` fields
4. Validate with a test conversation and conversation view

Inspect gateway traffic or failure:

1. `GET /api/v1/conversations?offsetminutes=<minutes>&filter=--gateway <gateway-id>`
2. `GET /api/v1/conversations/<conversation-id>`
3. Inspect resources, error message, messages, tools, usage, and model name

## RAG Intents

Inspect collections:

1. `GET /api/v1/collections`
2. `GET /api/v1/collections/<collection-id>`

Inspect retrieval:

1. `POST /api/v1/query`
2. `GET /api/v1/collections/<collection-id>/transactions?view=recent`
3. `GET /api/v1/collections/<collection-id>/transactions/<transaction-id>`

Find missing documents:

1. `GET /api/v1/collections/<collection-id>/documents?filter=<text>`
2. `GET /api/v1/collections/<collection-id>/documents?state=queued`
3. `GET /api/v1/collections/<collection-id>/documents?order_by=updated_at_desc`

## Batch Intents

Inspect workflows and jobs:

1. `GET /api/v1/batch/workflows`
2. `GET /api/v1/batch/workflows/<workflow-id>`
3. `GET /api/v1/batch/workflows/<workflow-id>/jobs`
4. `GET /api/v1/batch/workflows/<workflow-id>/jobs/<job-id>`

Inspect failed items:

1. `GET /api/v1/batch/workflows/<workflow-id>/jobs/<job-id>/items?state=ExecutionError`
2. `GET /api/v1/batch/workflows/<workflow-id>/jobs/<job-id>/items?confidence=low`
3. `GET /api/v1/batch/workflows/<workflow-id>/jobs/<job-id>/items/<item-id>`

## Cost Intents

Get current health:

1. `GET /api/v1/information/balance`

Break down usage:

1. `GET /api/v1/information/usage?timeStart=<iso-start>&timeEnd=<iso-end>`
2. Map top resource IDs to gateways, collections, chat clients, conversations, or batch jobs

## Conversation And Chat Intents

Analyze one conversation:

1. `GET /api/v1/conversations/<conversation-id>`
2. Inspect linked gateway/client/session/resources

Analyze bulk conversations:

1. `GET /api/v1/conversations?offsetminutes=<minutes>&filter=<filter>`
2. Export only when the listing is insufficient

Inspect chat client:

1. `GET /api/v1/web-chat-client`
2. `GET /api/v1/web-chat-client/<client-id>`
3. `GET /api/v1/web-chat-client/<client-id>/sessions`


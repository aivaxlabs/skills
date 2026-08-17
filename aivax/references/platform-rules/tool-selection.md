# Tool Selection

How to map an intent to a tool call. This file is a decision guide, not an endpoint catalog. Endpoint shapes live in `aivax_search_context` and `inference.aivax.net/apidocs/llms.txt`; this file decides which tool or endpoint to reach for first.

## General Decision Order

1. **Is the intent a model, RAG, rerank, classify, segment, speech, image, voice, multimodal, or test operation?** Load the matching capability sub-skill. It owns the call shape.
2. **Is the intent a gateway, batch, web chat, skill, account, cost, or observability operation?** Load the matching capability sub-skill.
3. **Is the intent a cross-cutting concern** (mutation safety, error interpretation, rate limit, idempotency)? Stay here.
4. **Do not know the field name, enum, or endpoint shape?** Call `aivax_search_context` (MCP) or fetch `inference.aivax.net/apidocs/llms.txt` (fallback) before guessing.

## Tool Choice

| Need | MCP | Fallback |
| --- | --- | --- |
| List or inspect account resources | `aivax_invoke_function` with `GET /api/v1/<resource>` | `GET https://inference.aivax.net/api/v1/<resource>` |
| Mutate account resources | `aivax_invoke_function` with `POST` / `PUT` / `PATCH` / `DELETE` | Same HTTPS method, JSON body, `Authorization: Bearer` |
| Discover or compare models | `aivax_list_models` (with `name_filter` for focus) | `GET https://inference.aivax.net/v1/models` |
| Verify field, enum, or behavior | `aivax_search_context` with `search_type: "all"` or `"api-function-reference"` | Fetch `https://inference.aivax.net/apidocs/llms.txt` or `https://docs.aivax.net/docs/<topic>` |
| Direct chat completion | Not available via MCP; use HTTP or OpenAI SDK with `base_url: https://inference.aivax.net/v1` | Same |
| Realtime voice | Not available via MCP; use HTTP `GET /v1/realtime-voice/sessions` | Same |

## Resource Path Map

Hostless paths, used through `aivax_invoke_function` or as the suffix of `https://inference.aivax.net`:

```text
GET  /api/v1/information/balance
GET  /api/v1/information/usage?timeStart=<iso>&timeEnd=<iso>
GET  /api/v1/information/rerankers
GET  /api/v1/information/audio-transcription

GET  /api/v1/ai-gateways
GET  /api/v1/ai-gateways/<gateway-id>
PATCH /api/v1/ai-gateways/<gateway-id>

GET  /api/v1/collections
GET  /api/v1/collections/<collection-id>
GET  /api/v1/collections/<collection-id>/documents
GET  /api/v1/collections/<collection-id>/transactions
GET  /api/v1/collections/<collection-id>/performances
PUT  /api/v1/collections/<collection-id>
PUT  /api/v1/collections/<collection-id>/documents
POST /api/v1/collections/<collection-id>/documents (JSONL import)
DELETE /api/v1/collections/<collection-id>/vectors-only

POST /api/v1/query
POST /api/v1/answer

GET  /api/v1/skills
GET  /api/v1/skills/<skill-id>
POST /api/v1/skills
PUT  /api/v1/skills/<skill-id>
DELETE /api/v1/skills/<skill-id>
GET  /api/v1/skills/management/export.jsonl
POST /api/v1/skills/management/import

GET  /api/v1/batch/workflows
GET  /api/v1/batch/workflows/<workflow-id>
GET  /api/v1/batch/workflows/<workflow-id>/jobs
GET  /api/v1/batch/workflows/<workflow-id>/jobs/<job-id>
GET  /api/v1/batch/workflows/<workflow-id>/jobs/<job-id>/items
POST /api/v1/batch/workflows
PATCH /api/v1/batch/workflows/<workflow-id>
POST /api/v1/batch/workflows/<workflow-id>/jobs
PATCH /api/v1/batch/workflows/<workflow-id>/jobs/<job-id>

GET  /api/v1/conversations
GET  /api/v1/conversations/<conversation-id>
DELETE /api/v1/conversations/<conversation-id>
GET  /api/v1/conversations/management/export.jsonl

GET  /api/v1/web-chat-client
GET  /api/v1/web-chat-client/<client-id>
POST /api/v1/web-chat-client
PUT  /api/v1/web-chat-client/<client-id>
GET  /api/v1/web-chat-client/<client-id>/sessions
POST /api/v1/web-chat-client/<client-id>/sessions

GET  /api/v1/agentic-tests
GET  /api/v1/agentic-tests/<test-id>
POST /api/v1/agentic-tests
PATCH /api/v1/agentic-tests/<test-id>
DELETE /api/v1/agentic-tests/<test-id>
POST /api/v1/agentic-tests/<test-id>/evaluate
POST /api/v1/agentic-tests/<test-id>/run
GET  /api/v1/agentic-tests/<test-id>/runs
GET  /api/v1/agentic-tests/<test-id>/runs/<run-id>
POST /api/v1/agentic-tests/<test-id>/runs/<run-id>/cancel

POST /api/v1/generations/rerank
POST /api/v1/generations/segment-text
POST /api/v1/generations/classify-documents
POST /api/v1/generations/describe-media
POST /api/v1/generations/speech
POST /api/v1/generations/images
POST /api/v1/generations/transcribe-audio

GET  /v1/realtime-voice/sessions

GET  /api/v1/keys
POST /api/v1/keys
DELETE /api/v1/keys/<key-id>

PUT  /api/v1/account
POST /api/v1/account/salt/roll
```

## When To Use The MCP And When To Use Direct HTTP

Use the MCP when:

- The user is operating a single AIVAX account interactively.
- The agent has the Account Management MCP configured.
- The work is inspection, mutation, or audit of account resources.

Use direct HTTP when:

- The user is integrating AIVAX as a service and the agent is writing the integration code.
- The work is inference, generation, or realtime voice (these are not exposed by the MCP).
- The MCP is unavailable but the public API and an API key are configured.

When both work, prefer the MCP for inspection and mutation, and direct HTTP for inference and realtime voice. The two paths must not be mixed within a single resource operation; pick one and stay consistent.

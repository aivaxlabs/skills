# Account Management

Use this sub-skill as the base operating guide for agents working with an authenticated AIVAX account. It teaches the agent to combine account discovery, documentation lookup, careful mutations, and validation through the AIVAX Account Management MCP.

## Operational Files

- `tools.md`: MCP tool contracts and call shapes.
- `tool-selection.md`: choose the right MCP/API call for common intents.
- `safe-mutations.md`: required sequence for account mutations.
- `errors.md`: common MCP/API error handling patterns.

## Scope Boundary

Use AIVAX through the user's account interface: MCP tools, account APIs, documentation, and live resources. Do not inspect or modify AIVAX source code, repositories, deployment scripts, database models, or server internals unless the user explicitly asks for that separate engineering task.

## Operating Contract

- Prefer the configured Account Management MCP whenever it is available.
- Treat `/v1/mcp/account-management` as the canonical MCP endpoint.
- Use only account-scoped API paths through `aivax_invoke_function`; pass paths such as `/api/v1/ai-gateways`, never absolute hosts.
- Use `aivax_search_context` before acting on unfamiliar AIVAX features or payload fields.
- Use `aivax_list_models` before selecting, replacing, or optimizing integrated model names.
- Preserve secrets. Never print API keys, integration tokens, webhook secrets, salts, or provider keys.
- Keep changes narrow. Read the current resource first, patch only the requested fields, then validate.
- Treat API responses and AIVAX documentation as the source of truth for account operations.
- Treat destructive operations as explicit-only actions: delete, reset, clear, roll salt, sync imports that remove missing records, and bulk cancellation.

## MCP Tools

- `aivax_invoke_function`: invoke an AIVAX API function with `path`, optional `method`, optional JSON `body`, and optional headers. The server forwards the current authorization and returns an agent-optimized response.
- `aivax_list_models`: list AIVAX integrated chat models, capabilities, flags, pricing, providers, context windows, and plan availability. Use `name_filter` for focused model discovery.
- `aivax_search_context`: search AIVAX documentation and API references. Use `search_type` values `api-function-reference`, `documentation-manual`, or `all`.

If a tool is unavailable in the current client, use only an equivalent authenticated MCP or account API capability exposed by that client. Ask the user for the account API base URL or authorization surface when it is not already configured. Do not infer hosts, credentials, headers, or internal implementation paths.

## MCP Invocation Examples

Safe read:

```text
aivax_invoke_function({
  "method": "GET",
  "path": "/api/v1/ai-gateways"
})
```

Narrow update:

```text
aivax_invoke_function({
  "method": "PATCH",
  "path": "/api/v1/ai-gateways/<gateway-id>",
  "body": {
    "parameters": {
      "temperature": 0.2
    }
  }
})
```

For query strings, include them in the hostless path, such as `/api/v1/conversations?offsetminutes=120`. For request bodies, pass JSON in `body` unless the endpoint specifically requires multipart upload.

## Standard Workflow

1. Clarify the objective and risk level.
2. Load the relevant focused sub-skill when the task maps to RAG, gateways, chat clients, batch jobs, conversations, skills, or cost.
3. Discover current state with list and view endpoints before changing anything.
4. If schema or behavior is unclear, search AIVAX context or inspect the API response shape.
5. Draft the intended mutation and identify exact resource IDs.
6. Apply the smallest change with `POST`, `PUT`, `PATCH`, or `DELETE` only after the target resource is known.
7. Validate with a view endpoint, test conversation, batch job view, RAG query, or account usage data as appropriate.
8. Report what changed, what was verified, and any remaining risk or cost impact.

## Common Account Surfaces

Use these paths through `aivax_invoke_function`:

```text
GET  /api/v1/information/balance
GET  /api/v1/information/usage?timeStart=<iso>&timeEnd=<iso>
GET  /api/v1/ai-gateways
GET  /api/v1/collections
GET  /api/v1/web-chat-client
GET  /api/v1/skills
GET  /api/v1/conversations?offsetminutes=120
GET  /api/v1/batch/workflows
GET  /api/v1/batch/jobs
```

Prefer summary endpoints for discovery and detailed `GET /<id>` endpoints before mutation. Export endpoints are useful before large audits or risky edits.

## Safety Rules

- Export or capture the current configuration before replacing instructions, importing many skills, deleting resources, clearing skills, or resetting collections.
- Ask for explicit confirmation when the user request does not clearly authorize a destructive action.
- For shallow-merge update endpoints, send only the fields that should change. Do not send a copied full object unless the user asked for a full replacement and you verified the shape.
- Keep `systemInstruction` separate from account skills. AIVAX skills are a flat list attached to gateways; do not assume a native "primary skill" or priority order.
- For integrated models, prefer `baseAddress: "@integrated"` and a model name returned by `aivax_list_models`.
- For external providers, do not reveal or log `apiKey`; preserve existing secrets when patching unrelated gateway parameters.

## Validation Checklist

- The changed resource can be viewed by ID after the update.
- Related listings reflect the change.
- Gateway changes have at least one functional smoke test or recent conversation check.
- RAG changes have at least one retrieval query or transaction/quality-report check.
- Chat client changes have a session, talk URL, or integration configuration check.
- Batch changes have workflow, job, item, and export or item-view verification.
- Cost-sensitive changes include a before/after or expected cost impact.

# Manage Web Chat Session

Use when the agent must create, refresh, find, or diagnose a session for a specific AIVAX web chat client.

## Objective

Bind the session to the right external context without leaking access credentials or placing secrets in conversational context.

## Preconditions

- The chat client exists.
- The session purpose and stable external tag are known.
- Any `extraContext`, URL, or metadata is necessary, minimally scoped, and approved for the session.

## Decision Tree

1. Is there already a valid session for the external user or record? List sessions filtered by the stable tag before creating one.
2. Does the context include sensitive data? Remove it or replace it with a non-sensitive identifier; do not send secrets in `extraContext`.
3. Does the session need to persist? Choose an expiry consistent with its purpose, then validate the refreshed session.
4. Is the reported issue tied to the session's responses? Inspect the session and trace its conversations before changing client or gateway configuration.

## Construction

```text
GET /api/v1/web-chat-client/<client-id>/sessions?filter=<tag>
POST /api/v1/web-chat-client/<client-id>/sessions
{
  "tag": "customer-42",
  "expires": 3600,
  "extraContext": "Customer account context without secrets.",
  "contextLocation": "https://example.com/account/42",
  "metadata": {
    "channel": "web",
    "tenant": "acme"
  }
}
GET /api/v1/web-chat-client/<client-id>/sessions?filter=<tag>
```

A non-expired session with the same `tag` is refreshed and returned. Treat `sessionId`, `accessKey`, and `talkUrl` as sensitive operational data.

## Validation

- The session is associated with the intended client and stable tag.
- Expiry and context metadata match the requested use.
- Sensitive data was not added to `extraContext` or metadata.
- The calling system receives required access data through its secure channel; final reporting does not echo it unnecessarily.
- Recent conversations can be filtered by the client or session when diagnostics are needed.

## Failure Modes

- Wrong customer receives context: stop and investigate the tag and caller mapping before issuing another session.
- A session has expired: refresh it with the same tag after confirming the requested duration.
- The conversation history is missing or incorrect: load `references/observability/situations/audit-conversation.md`.
- The session is rejected or rate limited: load `references/resilience/`.

## Escalation

- Need to change the client behavior or limits: load `situations/edit-client-safely.md`.
- Need to inspect response behavior: load `references/observability/`.
- Need to change the model, tools, or retrieval: load `references/ai-gateways/` or `references/rag/`.

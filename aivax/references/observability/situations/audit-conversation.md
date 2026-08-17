# Audit Conversation

Use when the agent must review stored conversation records for compliance, quality, post-mortem analysis, or customer support.

## Objective

Produce an audit conclusion that is anchored in evidence, redacted of sensitive data, and reproducible from the same query parameters.

## Preconditions

- The user can identify the scope (a single conversation ID, a chat session, a time window, a gateway, or a customer).
- The agent has access to the account (MCP or direct HTTPS with a private key).
- The audit purpose is clear (compliance, quality, post-mortem, customer support).

## Decision Tree

1. What is the scope?
   - Single conversation: use the view endpoint.
   - Single session: list conversations filtered by chat session.
   - Time window: list conversations with `offsetminutes` or export a JSONL.
   - Gateway: list conversations filtered by gateway.
   - Customer: list conversations filtered by external user ID.
2. Is the audit a quality review or a compliance review?
   - Quality: focus on tool calls, RAG citations, error rates, response times.
   - Compliance: focus on data access, PII handling, retention, deletion, moderation events.
3. Is the audit a post-mortem or a customer-support ticket?
   - Post-mortem: full trace, root cause, fix, validation.
   - Customer support: focused excerpt, no internal IDs, redacted PII.
4. Is the audit destructive (e.g. deletion for privacy cleanup)? Confirm with the user before any destructive action.

## Construction

```text
1. List conversations
   GET /api/v1/conversations?offsetminutes=<window>&filter=<filter>

2. View a representative sample
   GET /api/v1/conversations/<id>

3. Inspect
   origin, modelName, requestId, tools, usageObject, resources, timestamps, errorMessage, messages, metadata

4. Classify
   - Successful: produced a non-empty assistant message.
   - Failed: produced an error message; classify the error class.
   - Refused: produced a refusal; capture the reason.
   - Incomplete: produced a partial response; capture the finish_reason.

5. Redact
   - Replace PII with placeholders.
   - Replace API key identifiers, external user IDs, access keys, and secrets with redacted versions.
   - Summarize hidden reasoning instead of pasting it.

6. Report
   - Scope (window, filters, count).
   - Patterns (error rate, tool failures, low scores, refusals, completions).
   - Representative IDs (a few, redacted).
   - Conclusion and recommended action.
```

## Validation

- The audit conclusion is anchored in evidence.
- Sensitive data is redacted.
- The trace ID is preserved.
- The audit is reproducible: same scope, same filters, same result.
- The report does not include full transcripts, media contents, or hidden reasoning unless the user explicitly needs that exact data.

## Limits

- The view endpoint may shorten very long message lists. Use the export endpoint for full audits.
- The export endpoint produces JSONL. The agent must choose the smallest `period`, `media`, and `thinking` modes that satisfy the task.
- Deletion is destructive. Use only when the user explicitly asks.

## Escalation

- The audit reveals a quality issue: load `references/agentic-tests/situations/design-test-case.md` and add a regression test.
- The audit reveals a compliance issue: load `references/account/situations/secret-hygiene.md` and rotate the affected credentials.
- The audit reveals a degradation: load `situations/diagnose-degradation.md`.
- The audit reveals a cost anomaly: load `references/cost-monitoring/situations/investigate-cost-spike.md`.

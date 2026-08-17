# Correlate Request Trace

Use when the agent must follow a request from the entry point (API, web chat, integration) through every stage and resource to its final response.

## Objective

Build a complete picture of a single request: who initiated it, which gateway handled it, which model was used, which tools were called, which RAG collections were queried, how long each stage took, and what the final response was.

## Preconditions

- The user can identify the request (a conversation ID, a request ID, an external user ID, a chat session ID, or a time window).
- The agent has access to the account (MCP or direct HTTPS with a private key).

## Decision Tree

1. What is the entry point? API call, web chat, messaging integration, or batch.
2. Find the conversation. The conversations listing accepts filters by gateway, chat client, chat session, model, user, or API key. Use the smallest filter that resolves the conversation.
3. View the conversation. The view endpoint returns the model, the resources, the messages, the tool calls, the usage, and the timestamps.
4. For each tool call, follow the result. The conversation may include the tool name, the arguments, and the result.
5. For each RAG reference, follow the collection and the transaction. The conversation may include the collection ID and the transaction ID.
6. For each error, classify it. Load `references/platform-rules/error-handling.md` if the class is unclear.
7. Build the trace. The trace is a list of stages, each with the stage name, the duration, the IDs, and the output.

## Construction

```text
1. List conversations
   GET /api/v1/conversations?offsetminutes=<window>&filter=<filter>

2. View the conversation
   GET /api/v1/conversations/<conversation-id>

3. Inspect
   origin, modelName, requestId, tools, usageObject, resources, timestamps, errorMessage, messages, metadata

4. Follow each resource
   gateway: GET /api/v1/ai-gateways/<id>
   collection: GET /api/v1/collections/<id>
   chat client: GET /api/v1/web-chat-client/<id>
   session: GET /api/v1/web-chat-client/<id>/sessions/<session-id>
   batch job: GET /api/v1/batch/workflows/<id>/jobs/<id>
   agentic test: GET /api/v1/agentic-tests/<id>/runs/<run-id>

5. Build the trace
   stages: [
     { name: "entry", ids: { conversation_id, request_id }, duration, output: "summary" },
     { name: "gateway", ids: { gateway_id, model_id }, duration, output: "summary" },
     { name: "rag", ids: { collection_id, transaction_id }, duration, output: "summary" },
     { name: "tool", ids: { tool_name, tool_call_id }, duration, output: "summary" },
     { name: "response", duration, output: "summary" }
   ]
```

## Validation

- The trace is anchored in evidence (the conversation view, the resource views).
- The stages are in the right order.
- The IDs match the resources the agent has access to.
- The error (if any) is classified correctly.
- The trace ID is preserved.

## Failure Modes

- The conversation is not in the listing: the filter is wrong, the window is too small, or the conversation belongs to another account. Adjust.
- The view endpoint shortens the messages: the conversation is long. Use the export endpoint for the full record.
- A resource is no longer accessible: it was deleted. The trace is incomplete; surface this.

## Limits

- Do not paste full transcripts, media, hidden reasoning, or private user data in the trace. Summarize.
- Some resources may be deleted; the trace is then incomplete.

## Escalation

- The error class is unclear: load `references/platform-rules/error-handling.md`.
- The RAG transaction is poor: load `references/rag/situations/debug-bad-answer.md`.
- The batch job failed: load `references/batch/situations/debug-failed-job.md`.
- The cost is unexpectedly high: load `references/cost-monitoring/situations/investigate-cost-spike.md`.

# Cross-Resource Debugging

Use this workflow when the user reports a failure but the affected AIVAX layer is unclear.

## Goal

Identify whether the issue belongs to gateway configuration, model capability, RAG retrieval, tool/MCP execution, chat client/session state, batch execution, account limits, or cost/balance.

## Steps

1. Capture the user-visible symptom, approximate time, channel, and expected behavior.
2. Identify the entry point:
   - API or OpenAI-compatible request
   - Chat client/session
   - Messaging integration
   - Batch job/item
3. List or view the relevant conversations, jobs, or resources.
4. Follow resource links back to gateway, model, skills, collections, tools, chat client, session, and usage.
5. Classify the failure layer.
6. Load the matching situation playbook.
7. Apply the smallest fix only after reading current state.
8. Verify through the original failure path.

## Evidence To Collect

- Conversation ID or batch item ID.
- Gateway ID and model name actually used.
- Error message and timestamp.
- Tool calls and tool results.
- RAG collections and transaction IDs.
- Chat client/session/tag when channel-specific.
- Usage/cost for slow or expensive cases.
- Recent configuration changes when available.

## Avoid

- Changing gateway parameters before viewing the gateway.
- Reindexing or resetting collections before checking retrieval evidence.
- Retrying all batch failures before sampling failed items.
- Switching models before checking model capability and cost.
- Exposing transcripts, access keys, external user IDs, or secrets in the final answer.


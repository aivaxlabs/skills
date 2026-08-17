# Diagnose Degradation

Use when a gateway, chat client, model, or batch job is slower than expected, erroring more than expected, or producing lower-quality output than expected.

## Objective

Identify the dominant cause of the degradation, connect it to a resource or a setting, and recommend a safe action — without changing anything before the user approves.

## Preconditions

- The user can describe the symptom (slow, erroring, low quality) and the time window.
- The agent has access to the account (MCP or direct HTTPS with a private key).
- The user can name the affected resource (a gateway, a chat client, a model, a batch workflow).

## Decision Tree

1. Get the recent conversations for the affected resource. The conversations listing accepts filters by gateway, chat client, chat session, model, user, or API key.
2. Inspect the timestamps, the token counts, the model name, the tool calls, the RAG transaction references, and the error messages.
3. For each potential cause, collect evidence:
   - **Model choice**: is the model the same as the one that was fast before? Has the model been swapped, deprecated, or routed differently?
   - **RAG**: are the collection transactions slower? Higher process time? Lower score? More retrieved documents?
   - **Tools**: are the tool calls more frequent? Larger? Failing? Looping?
   - **Context size**: has the session context grown? Is the `contextMaximumSize` lower than the recent context?
   - **Output length**: is the model emitting longer responses? Is `maxCompletionTokens` higher than needed?
   - **Provider behavior**: has the provider changed pricing, latency, or availability? Check the model catalog.
   - **Channel behavior**: has the chat client's session settings changed? Debounce, scheduled continuations, audio synthesis?
   - **Batch load**: is a batch job running that consumes subscription reserve? Pause or finish the job.
4. Classify the dominant cause. One cause usually dominates; the others are usually correlated.
5. Propose the smallest change. Order by safety and behavior risk.
6. Get explicit approval before any mutation.

## Common Causes

- Model changed to a slower or more expensive model.
- Long outputs or high `maxCompletionTokens`.
- RAG injected too many documents.
- Tool loop or repeated tool calls.
- Session context grew too large.
- External provider latency.
- Batch or subscription reserve pressure.
- Recent configuration change in the gateway, chat client, or session.

## Validation

- The dominant cause is supported by evidence (timestamps, token counts, tool counts, error messages).
- The recommended change is ordered by safety and behavior risk.
- The user has approved the change before any mutation.
- After the change, the next observation window reflects the expected delta.

## Limits

- Multiple causes can co-exist. Pick the dominant one and address the others in subsequent changes.
- Some degradations are external (provider, network). The agent can swap models but cannot fix the provider.
- A degradation may be the intended effect of a previous change. Verify the history before reverting.

## Escalation

- The cause is a model choice: load `references/text-inference/situations/choose-model.md`.
- The cause is a RAG pipeline: load `references/rag/situations/design-rag-pipeline.md`.
- The cause is a chat client or session: load `references/web-chat/`.
- The cause is a batch retry storm: load `references/batch/situations/debug-failed-job.md`.
- The cause is a cost spike: load `references/cost-monitoring/situations/investigate-cost-spike.md`.

# Cross-Skill Observability

Use this situation when a composed pipeline runs and the agent needs to keep trace IDs, request IDs, and resource IDs correlated across stages.

## Objective

Make the pipeline auditable from end to end. A reviewer (or a future agent) should be able to look at a single trace ID and see every stage's inputs, outputs, IDs, and durations.

## Preconditions

- A pipeline is being executed. The pipeline has stages, each with a name and an owning sub-skill.
- The pipeline has a trace ID generated at start.

## Signals

- A failure happened and the cause is unclear.
- The user asks "why was this slow / expensive / wrong?"
- The agent needs to compare two pipeline runs.
- A new stage is being substituted and the agent needs to confirm the trace still propagates.

## Trace Propagation Rules

1. Generate a single `trace_id` at pipeline start. Use a UUID or a timestamp-prefixed string.
2. Pass `trace_id` to every stage. Each stage includes it in `metadata` when the underlying call supports metadata (chat completions support `metadata` for this purpose; for other calls, store it in the agent's working state).
3. Each stage records its own resource IDs as the stage runs:
   - `model_id` (or gateway_id) for inference stages.
   - `collection_id` and `transaction_id` for RAG stages.
   - `reranker` and `request_id` from the rerank response for rerank stages.
   - `conversation_id` for any chat-completion call.
   - `item_id` and `job_id` for batch stages.
   - `run_id` for agentic-test stages.
4. Each stage records its duration and a cost estimate (token usage, generation context, or flat cost).
5. The final report is a list of stages, each with: stage name, sub-skill, input summary, output summary, IDs, duration, cost, and any error class.

## Report Format

```text
trace_id: tr_<id>
started: <iso>
finished: <iso>
total_cost: <estimate>
total_latency: <ms>

stages:
  - name: classify
    skill: references/text-tools/
    duration: 120ms
    cost: 0.0001
    ids: { request_id: req_abc }
    inputs: <summary>
    outputs: <summary>
  - name: retrieve
    skill: references/rag/
    duration: 240ms
    cost: 0.0003
    ids: { collection_id: c_123, transaction_id: t_456 }
  - name: rerank
    skill: references/rerankers/
    duration: 180ms
    cost: 0.0005
    ids: { reranker: "@aivax/reflex-v1", request_id: req_def }
  - name: generate
    skill: references/text-inference/
    duration: 1800ms
    cost: 0.012
    ids: { conversation_id: conv_xyz, request_id: req_ghi }
  - name: evaluate
    skill: references/agentic-tests/
    duration: 900ms
    cost: 0.004
    ids: { test_id: at_42, run_id: run_99 }
```

## Validation

- The trace ID is the same in every stage's record.
- The total cost is the sum of stage costs (with rounding tolerance).
- The total latency is at least the sum of stage durations (network and orchestration overhead add to it).
- A failure in one stage is visible in the report with the stage name, error class, and the IDs that were issued before the failure.

## Escalation

- A stage does not support metadata: store the trace ID in the agent's working state and surface it in the report, but do not attempt to inject it into a call that does not accept it.
- A stage's IDs are not visible in the response: fall back to the agent's pre-call state and the next stage's input. Do not invent IDs.
- The pipeline is too long to keep a clean trace (>20 stages): split into a parent trace and child traces, with the parent trace ID propagated to the children.

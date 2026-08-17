# Debug Failed Batch Job

Use when a job is paused, failing, producing validation errors, returning low-confidence results, or costing more than expected.

## Objective

Identify whether the fault is input-specific or systemic before changing a workflow, retrying items, or spending more credits.

## Preconditions

- Workflow and job identifiers are known.
- The job and a representative set of affected items are accessible.
- The agent can distinguish execution failures, validation failures, low confidence, and high cost from the job data.

## Decision Tree

1. Are failures isolated to a few inputs? Inspect those inputs and their item details before changing the workflow.
2. Are failures systemic? Compare the workflow's model, schema, validation, tools, retries, and input shape across affected items.
3. Is the job paused due to cost or balance? Inspect balance and usage before resuming.
4. Is the issue a provider or rate-limit failure? Use the narrow execution-error retry after loading `references/resilience/`.
5. Is output useful but validation fails? Align schema and validation instruction rather than increasing retries.

## Investigation

```text
GET /api/v1/batch/workflows/<workflow-id>
GET /api/v1/batch/workflows/<workflow-id>/jobs/<job-id>
GET /api/v1/batch/workflows/<workflow-id>/jobs/<job-id>/items?state=ExecutionError
GET /api/v1/batch/workflows/<workflow-id>/jobs/<job-id>/items?state=ValidationError
GET /api/v1/batch/workflows/<workflow-id>/jobs/<job-id>/items?confidence=low
GET /api/v1/batch/workflows/<workflow-id>/jobs/<job-id>/items/<item-id>
GET /api/v1/information/balance
GET /api/v1/information/usage?timeStart=<iso>&timeEnd=<iso>
```

Capture the item input shape, output, validation result, error message, confidence, state, retry count, model or gateway, enabled tools, and per-item versus total cost.

## Common Causes

- Input lines are malformed JSONL, use an unsupported import shape, or omit stable record IDs.
- The result schema is too strict or contradicts the instruction.
- The validation instruction rejects useful output or asks for evidence absent from the input.
- The model or gateway is unavailable or too weak for the task.
- A required tool fails repeatedly.
- Retry policy is too low to recover transient faults or too high for deterministic faults.
- The job was scaled without a pilot.

## Correction And Validation

1. Pause the job when failures or cost are ongoing.
2. Apply the smallest confirmed correction: fix inputs, adjust the schema or instruction, change model only with evidence, or resolve a tool/provider fault.
3. Run a small representative sample.
4. Retry the narrowest matching mode:

```text
POST /api/v1/batch/workflows/<workflow-id>/jobs/<job-id>/items/retry?mode=execution-error
POST /api/v1/batch/workflows/<workflow-id>/jobs/<job-id>/items/retry?mode=validation-error
POST /api/v1/batch/workflows/<workflow-id>/jobs/<job-id>/items/retry?mode=low-confidence
```

5. Confirm the new items complete with valid output, acceptable confidence, and expected per-item cost.

## Failure Modes

- Retrying all errors without diagnosis repeats deterministic failures and raises cost.
- Editing a workflow while attributing existing failures to the new configuration confuses the investigation; record the change point.
- Removing items to hide a problem is destructive; require explicit approval and preserve an export or evidence first.

## Escalation

- Need model selection: load `references/text-inference/situations/choose-model.md`.
- Need gateway changes: load `references/ai-gateways/situations/edit-gateway-safely.md`.
- Need cost control: load `references/cost-monitoring/situations/investigate-cost-spike.md`.
- Need transient failure handling: load `references/resilience/`.
- Need tracing: load `references/observability/`.

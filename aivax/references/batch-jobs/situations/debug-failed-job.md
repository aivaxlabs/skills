# Debug Failed Batch Job

Use when a batch job is paused, failing, producing validation errors, has low confidence, or costs more than expected.

Goal: identify whether the failure is workflow design, model/gateway availability, input shape, schema, validation instruction, tool use, retry policy, balance, or active job state.

## Steps

1. View workflow:

```text
GET /api/v1/batch/workflows/<workflow-id>
```

2. View job:

```text
GET /api/v1/batch/workflows/<workflow-id>/jobs/<job-id>
```

3. Inspect failed and low-confidence items:

```text
GET /api/v1/batch/workflows/<workflow-id>/jobs/<job-id>/items?state=ExecutionError
GET /api/v1/batch/workflows/<workflow-id>/jobs/<job-id>/items?confidence=low
```

4. View representative item details.
5. Check `modelName`, result schema, validation instruction, enabled tools, retry limits, item input shape, and total cost.
6. If balance is suspicious, inspect account balance and usage.
7. Fix the workflow only after confirming whether the issue is item-specific or systemic.

## Common Causes

- Input lines are not valid JSONL or lack stable IDs.
- Result schema is too strict or mismatched with instructions.
- Validation instruction rejects otherwise useful output.
- Model/gateway unavailable or too weak for the task.
- Tool call failed for many items.
- Retry limits are too low or too high.
- Job was started at full scale without a pilot.

## Validation

- Run or inspect a small representative sample.
- Retry targeted failed modes, not all items blindly.
- Confirm new finished items have valid output, acceptable confidence, and expected per-item cost.


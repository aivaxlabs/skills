# Run And Monitor Batch Job

Use when the workflow exists and the agent must create a job, import items, start a pilot or full run, monitor it, and export results.

## Objective

Move from verified input to a controlled execution, catching systemic errors and unexpected spend before they affect the full workload.

## Preconditions

- The workflow is inspected and suitable for the intended inputs.
- Inputs have stable IDs and the selected import mode is known.
- Balance and expected pilot cost are acceptable.
- The job scope and start approval are clear.

## Decision Tree

1. Is this the first run of a workflow or a materially changed dataset? Start with 5–20 representative items.
2. What import mode matches the source? Use `lines` for one non-empty line per record, `files` for plain-text files, `zip` for plain-text ZIP entries, or `text` for one text payload.
3. Are the imported item count and sampled inputs correct? If not, fix the import before activation.
4. Does the pilot have execution errors, validation errors, low confidence, or high per-item cost? Pause and diagnose before scaling.
5. Does the export need only finished/high-confidence output or an error review set? Apply the narrowest filters.

## Construction

```text
POST /api/v1/batch/workflows/<workflow-id>/jobs
{
  "title": "July import"
}

POST /api/v1/batch/workflows/<workflow-id>/jobs/<job-id>/items
# Multipart form data: mode=lines and items=<input file>

GET /api/v1/batch/workflows/<workflow-id>/jobs/<job-id>
GET /api/v1/batch/workflows/<workflow-id>/jobs/<job-id>/items?limit=100

PATCH /api/v1/batch/workflows/<workflow-id>/jobs/<job-id>
{
  "state": "Active"
}
```

New jobs start paused. For JSONL, use one compact UTF-8 JSON object per line. The multipart field is `items`; `documents` is accepted as an alias. If the available client cannot send multipart data, ask for the supported upload surface instead of inventing a JSON replacement.

Monitor with:

```text
GET /api/v1/batch/workflows/<workflow-id>/jobs/<job-id>
GET /api/v1/batch/workflows/<workflow-id>/jobs/<job-id>/items?state=ExecutionError
GET /api/v1/batch/workflows/<workflow-id>/jobs/<job-id>/items?state=ValidationError
GET /api/v1/batch/workflows/<workflow-id>/jobs/<job-id>/items?confidence=low
```

Pause with `{"state":"Paused"}`. Terminate with `{"state":"Finished"}` only when the job should not resume.

Export after inspection:

```text
GET /api/v1/batch/workflows/<workflow-id>/jobs/<job-id>/export.jsonl?state=finished
GET /api/v1/batch/workflows/<workflow-id>/jobs/<job-id>/export.jsonl?state=errors
GET /api/v1/batch/workflows/<workflow-id>/jobs/<job-id>/export.jsonl?state=finished&confidence=high
```

## Validation

- Imported count and sampled inputs match the source.
- Pilot results satisfy the schema and validation requirements.
- Error, validation, low-confidence, and per-item-cost rates are acceptable before scale-up.
- Job state matches the intended operation.
- Exported JSONL joins back to source records by stable ID.

## Failure Modes

- Import count is wrong: pause and correct the source or mode before starting.
- Cost rises unexpectedly: pause, collect job and item evidence, then load `references/cost-monitoring/`.
- Repeated item failures: load `situations/debug-failed-job.md`; do not blindly retry all items.
- Job is rate limited or has a transient provider failure: load `references/resilience/`.

## Escalation

- Need targeted retries or root-cause investigation: load `situations/debug-failed-job.md`.
- Need to change the workflow: load `situations/design-batch-workflow.md`.
- Need trace-level evidence: load `references/observability/`.

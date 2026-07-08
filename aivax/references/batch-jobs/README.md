# Batch Jobs

Use this sub-skill to operate AIVAX asynchronous batch processing. A batch workflow defines how each item should be processed; a batch job is one execution queue under that workflow; batch items are the individual inputs and outputs.

This is for designing and operating batch workflows/jobs through AIVAX account APIs. Do not inspect or modify AIVAX batch worker source code or queue implementation unless the user explicitly asks for source-code work.

## Situation Playbooks

- `situations/debug-failed-job.md`: paused, failing, validation-error, low-confidence, or expensive batch job.

Use the listed API paths through `aivax_invoke_function`. Search AIVAX context before changing uncertain workflow schemas, import modes, retry behavior, or export options.

```text
aivax_invoke_function({
  "method": "GET",
  "path": "/api/v1/batch/workflows"
})

aivax_invoke_function({
  "method": "PATCH",
  "path": "/api/v1/batch/workflows/<workflow-id>",
  "body": {
    "maxRetries": 1
  }
})
```

## Design Workflow

Before creating anything, define:

- Goal: what each item should transform, classify, extract, summarize, validate, or enrich.
- Input unit: one JSON object, one text line, one file, or one record per item.
- Output contract: a JSON schema that successful outputs must satisfy.
- Model or gateway: a model/gateway slug available to the account.
- Validation instruction: optional judge criteria for confidence and rejection.
- Tool requirements: built-in functions needed during item processing.
- Failure policy: `errorStopThreshold` and `maxRetries`.
- Cost expectations: approximate item count, model price, token volume, and validation overhead.

Use `aivax_list_models` or gateway listing before selecting `modelName`.

## Workflow Discovery

```text
GET /api/v1/batch/workflows
GET /api/v1/batch/workflows?filter=<text>
GET /api/v1/batch/workflows/<workflow-id>
GET /api/v1/batch/jobs
```

If a similar workflow exists, prefer editing or reusing it instead of creating duplicates.

## Create a Workflow

```text
POST /api/v1/batch/workflows
{
  "title": "Lead Qualification",
  "instruction": "Analyze the input lead record and return structured qualification data.",
  "resultSchema": {
    "type": "object",
    "additionalProperties": false,
    "required": ["qualified", "reason"],
    "properties": {
      "qualified": { "type": "boolean" },
      "reason": { "type": "string" },
      "confidence": { "type": "number" }
    }
  },
  "validationInstruction": "Reject outputs that do not justify the qualification decision from the input.",
  "modelName": "<model-or-gateway-slug>",
  "errorStopThreshold": 5,
  "maxRetries": 2
}
```

`modelName` must resolve to a model or gateway available to the account.

## Edit a Workflow

Use `PATCH /api/v1/batch/workflows/<workflow-id>` with only fields to change:

```text
PATCH /api/v1/batch/workflows/<workflow-id>
{
  "validationInstruction": "Return low confidence when required source data is missing.",
  "maxRetries": 1
}
```

`enabledTools` is shallow-merged. Keep tool enablement minimal because tools increase cost, latency, and failure surface.

## Create and Load a Job

Create a job. New jobs start paused:

```text
POST /api/v1/batch/workflows/<workflow-id>/jobs
{
  "title": "July import"
}
```

Import items:

```text
POST /api/v1/batch/workflows/<workflow-id>/jobs/<job-id>/items
```

Send multipart form data with:

- `mode=lines`: each non-empty line in `items` or `documents` becomes one item.
- `mode=files`: each uploaded text file becomes one item.
- `mode=zip`: each plain-text ZIP entry becomes one item.
- `mode=text`: the `text` field becomes one item.

For JSONL inputs, keep one compact UTF-8 JSON object per line. Include stable record IDs inside the input so exported outputs can be joined back later. For `mode=lines`, prefer the field name returned by API docs or current client examples; common upload fields are `items` or `documents`. If the current MCP client cannot send multipart form data, ask for the supported upload surface instead of inventing a JSON replacement.

Start processing only after checking item count and expected cost:

```text
PATCH /api/v1/batch/workflows/<workflow-id>/jobs/<job-id>
{
  "state": "Active"
}
```

For large jobs, run a pilot first: import 5 to 20 representative items, activate the job, review finished outputs, validation results, error rate, and cost, then scale only when results are acceptable. Pause before scaling if execution errors, validation errors, or low-confidence items appear repeatedly, or if actual per-item cost is materially above the estimate.

Pause or finish with the same endpoint:

```text
PATCH /api/v1/batch/workflows/<workflow-id>/jobs/<job-id>
{
  "state": "Paused"
}
```

## Monitor Jobs

```text
GET /api/v1/batch/workflows/<workflow-id>/jobs
GET /api/v1/batch/workflows/<workflow-id>/jobs/<job-id>
GET /api/v1/batch/workflows/<workflow-id>/jobs/<job-id>/items
GET /api/v1/batch/workflows/<workflow-id>/jobs/<job-id>/items?state=ExecutionError
GET /api/v1/batch/workflows/<workflow-id>/jobs/<job-id>/items?confidence=low
GET /api/v1/batch/workflows/<workflow-id>/jobs/<job-id>/items/<item-id>
```

Review `Summary`, `Progression`, `LatestItems`, item `state`, `confidence`, `validationResult`, and `totalCost`.

## Retry and Cleanup

Retry only non-running failed or low-confidence items:

```text
POST /api/v1/batch/workflows/<workflow-id>/jobs/<job-id>/items/retry?mode=errors
POST /api/v1/batch/workflows/<workflow-id>/jobs/<job-id>/items/retry?mode=execution-error
POST /api/v1/batch/workflows/<workflow-id>/jobs/<job-id>/items/retry?mode=validation-error
POST /api/v1/batch/workflows/<workflow-id>/jobs/<job-id>/items/retry?mode=low-confidence
```

Bulk removal is destructive. Use only when explicitly requested:

```text
DELETE /api/v1/batch/workflows/<workflow-id>/jobs/<job-id>/items?mode=pending
DELETE /api/v1/batch/workflows/<workflow-id>/jobs/<job-id>/items?mode=finished
DELETE /api/v1/batch/workflows/<workflow-id>/jobs/<job-id>/items?mode=errors
DELETE /api/v1/batch/workflows/<workflow-id>/jobs/<job-id>/items?mode=all
```

## Export Results

```text
GET /api/v1/batch/workflows/<workflow-id>/jobs/<job-id>/export.jsonl?state=all
GET /api/v1/batch/workflows/<workflow-id>/jobs/<job-id>/export.jsonl?state=errors
GET /api/v1/batch/workflows/<workflow-id>/jobs/<job-id>/export.jsonl?confidence=low
```

Exports include metadata, input, output, validation result, and confidence. Pending items are not exported.

## Troubleshooting

- Job paused for insufficient funds: inspect cost monitoring.
- Many `ExecutionError` items: inspect item input, model/gateway availability, tool errors, and retry limits.
- Many `ValidationError` items: simplify the result schema, align validation instruction with schema, or improve input normalization.
- Low confidence but valid JSON: tighten instruction or add examples.
- High cost per item: lower output length, choose a cheaper model, disable unnecessary tools, or split validation into a smaller model/gateway.

## Validation

Verify workflow view, job view, a small finished sample item, exported JSONL shape, and expected account usage categories before scaling up.

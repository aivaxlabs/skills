---
name: aivax-batch
description: Design and operate AIVAX asynchronous batch workflows, jobs, imports, monitoring, retries, cleanup, and exports. Load when the same AI operation must run independently across many records or when a batch job needs investigation.
---

# Batch

Batch runs the same AI workflow over many independent items in the background. A workflow defines instructions, model, output schema, validation, tools, and retry policy. A job is one execution queue under that workflow. Items hold individual input, output, validation, state, confidence, and cost.

## Operating Files

- `situations/design-batch-workflow.md`: define a reusable workflow and validate it on a representative pilot.
- `situations/run-and-monitor-job.md`: create a paused job, import items, start it deliberately, and monitor progress.
- `situations/debug-failed-job.md`: diagnose execution, validation, confidence, input, cost, and retry failures.

## When To Use Batch

Use Batch when dozens to thousands of independent records need the same classification, extraction, enrichment, summarization, moderation, structured generation, or tool-assisted processing.

Do not use Batch when inputs need shared memory, when a user needs an immediate single answer, or when the workflow is mainly searchable knowledge. Use direct inference for one immediate result and `references/rag/` for retrieval over a knowledge base.

## Operating Surfaces

- `GET` / `POST /api/v1/batch/workflows`: list or create workflows.
- `GET` / `PATCH /api/v1/batch/workflows/<workflow-id>`: inspect or update a workflow.
- `GET /api/v1/batch/workflows/<workflow-id>/jobs`: list a workflow's jobs.
- `POST /api/v1/batch/workflows/<workflow-id>/jobs`: create a paused job.
- `GET` / `PATCH /api/v1/batch/workflows/<workflow-id>/jobs/<job-id>`: inspect, start, pause, or finish a job.
- `POST /api/v1/batch/workflows/<workflow-id>/jobs/<job-id>/items`: import items as multipart data.
- `GET /api/v1/batch/workflows/<workflow-id>/jobs/<job-id>/items`: list and filter items.
- `POST /api/v1/batch/workflows/<workflow-id>/jobs/<job-id>/items/retry?mode=<mode>`: retry matching non-running items.
- `DELETE /api/v1/batch/workflows/<workflow-id>/jobs/<job-id>/items?mode=<mode>`: remove matching non-running items (destructive).
- `GET /api/v1/batch/workflows/<workflow-id>/jobs/<job-id>/export.jsonl`: export completed results.

Use `aivax_search_context` before changing uncertain workflow schemas, import fields, retry behavior, or export filters.

## Design Contract

Define these before creating a workflow:

- **Goal:** the same operation every item should perform.
- **Input unit:** one JSON object, text line, plain-text file, ZIP entry, or text payload per item.
- **Output contract:** JSON Schema for successful outputs.
- **Model or gateway:** available to the account and suitable for the quality, cost, and latency target.
- **Validation:** optional criteria for output correctness and confidence.
- **Tools:** only the tools required for every item.
- **Failure policy:** `errorStopThreshold` and `maxRetries`.
- **Cost expectation:** item count, expected tokens, model price, validation overhead, and a pilot budget.

Use stable record identifiers in input JSONL so exported outputs can be joined back to the source system.

## Import, State, and Retry Rules

Jobs begin **paused**. Import and inspect items before spending credits. Import modes are `lines`, `files`, `zip`, and `text`; `lines` is the default. Empty lines are skipped. Use multipart field `items` (`documents` is accepted as an alias) for line imports. Plain-text files only are accepted for file and ZIP imports.

Use `Active` to start a job, `Paused` to stop it temporarily, and `Finished` to terminate it. Pilot 5–20 representative items before processing the full workload.

Item states include `Pending`, `Finished`, `Refused`, `ExecutionError`, `ValidationError`, and `Cancelled`. Retry modes are `errors`, `execution-error`, `validation-error`, and `low-confidence`. A retry moves matching non-running items to `Pending` and automatically starts a non-active job if at least one item is retried.

## Safety And Cost Control

- Inspect balance and estimate cost before a large run.
- Start only after the imported item count and sample inputs look correct.
- Retry the narrowest mode justified by evidence; do not retry all items blindly.
- `DELETE` only removes non-running items but is still destructive. Require explicit approval.
- Export only the necessary state and confidence subset; pending items are not exported.

## Validation

- Workflow model, schema, validation, tools, and retries match the intended contract.
- A pilot produces valid outputs with acceptable confidence and per-item cost.
- The full job progresses without systemic error or validation failures.
- Exports preserve source record IDs and contain the expected output and metadata.

## Escalation

- Need model or gateway selection: load `references/text-inference/` or `references/ai-gateways/`.
- Need batch-scale audio, image, text, or RAG processing: load the corresponding capability skill, then return here for execution.
- Need transient failure or 429 handling: load `references/resilience/`.
- Need cost analysis: load `references/cost-monitoring/`.
- Need request-level diagnosis: load `references/observability/`.

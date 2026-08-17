# Design Batch Workflow

Use when the same operation must process many independent records and the agent must define or create the reusable Batch workflow.

## Objective

Create a clear, testable recipe that produces joinable, validated outputs at a predictable cost before it reaches the full dataset.

## Preconditions

- Items are independent; they do not need shared memory or sequence-dependent decisions.
- The input unit, success criteria, and downstream output consumer are known.
- A model or gateway available to the account has been selected.
- A representative pilot dataset is available.

## Decision Tree

1. Is Batch the right surface? Use direct inference for one immediate answer and RAG for searchable knowledge.
2. Can each input carry a stable source ID? Add one before running, so exports can be joined back.
3. Is structured output required? Define a minimal JSON Schema with only the required fields.
4. Is validation necessary? Add it when a wrong result is costly or hard to detect downstream; account for its cost.
5. Are tools necessary for every item? Enable only the ones needed for the workflow.
6. Is the model proven? Test the prompt and schema on a small representative pilot before scaling.

## Construction

```text
GET /api/v1/batch/workflows?filter=<similar-workflow>
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
GET /api/v1/batch/workflows/<workflow-id>
```

Reuse or minimally edit an existing matching workflow rather than creating duplicates. Verify unfamiliar fields with `aivax_search_context`.

## Validation

- Workflow instruction has one consistent job for each item.
- Schema accepts the intended useful result and rejects malformed output without requiring unsupported detail.
- Model or gateway is available and appropriate for the target cost/quality balance.
- Failure thresholds and retries reflect the risk of the operation.
- A 5–20 item pilot is ready through `situations/run-and-monitor-job.md`.

## Failure Modes

- Items require each other's context: Batch is the wrong surface; redesign the input or use a different capability.
- Schema is too strict: reduce needless requirements or align the instruction before raising retries.
- Validation rejects useful outputs: clarify the criteria and inspect samples before disabling validation.
- A gateway or model is unavailable: load `references/text-inference/situations/choose-model.md`.

## Escalation

- Need to execute the pilot or import a dataset: load `situations/run-and-monitor-job.md`.
- Need workflow diagnostics: load `situations/debug-failed-job.md`.
- Need to estimate or reduce costs: load `references/cost-monitoring/`.

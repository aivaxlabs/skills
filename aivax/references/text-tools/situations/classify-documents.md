# Classify Documents

Use when the agent must classify one or many documents through `POST /api/v1/generations/classify-documents`.

## Objective

Assign one or more labels to each document, with the right granularity, language coverage, and cost.

## Preconditions

- A classification model is available on the account. Verify with `aivax_list_models`.
- The labels are defined. Each label has a name and a short description.
- The documents are in a supported language and format.

## Decision Tree

1. How many labels? Few labels (1 to 5) are easier for the model. Many labels (>10) need stronger models and clear descriptions.
2. Are the labels exclusive (single-label) or non-exclusive (multi-label)? Verify the model's behavior with `aivax_search_context`.
3. Is the language known? Set `language` explicitly when known. Auto-detect only when the language is unknown.
4. Is the input large? Each item counts against the quota. Split large jobs into smaller requests or load `references/batch/`.
5. Is the classification deterministic? Set `temperature: 0` (or the model's equivalent) when the same input should produce the same output.

## Construction

```text
{
  "model": "<classification-model>",
  "documents": [
    "Document 1 text",
    "Document 2 text"
  ],
  "labels": ["billing", "support", "feature_request", "other"],
  "metadata": { "trace_id": "tr_..." }
}
```

Some models accept `language`, `temperature`, and `max_labels`. Verify with `aivax_search_context`.

## Response Handling

1. The response is a list of classification results, one per document.
2. Each result has the assigned label(s) and, when supported, a confidence score.
3. Use the result to route the document to the next stage of the pipeline.
4. Record the labels and the model's confidence in the trace.

## Validation

- The result matches the document at a glance. Spot-check a few.
- The label distribution is plausible. If 99% of documents are in one label, the labels may be too narrow or the model is over-fitting.
- The trace ID is preserved.

## Failure Modes

- The model returns no label: the input is empty or unsupported. Inspect the error.
- The model returns the wrong label: the labels are ambiguous or the model is not strong on the domain. Adjust the labels or switch the model.
- The model returns many labels: the labels are too broad. Narrow the set.

## Escalation

- The classification is part of a larger pipeline: load `references/composition/situations/design-pipeline.md`.
- The task is large: load `references/batch/`.
- The classification needs to be evaluated: load `references/agentic-tests/situations/evaluate-by-criteria.md`.

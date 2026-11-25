# Classify Documents

Use when the agent must classify one or many documents through `POST /api/v1/generations/classify`.

## Objective

Score every document against a fixed label set and pick the label or labels to act on, with the right label wording and cost.

## Preconditions

- The labels are defined as short, descriptive phrases. Labels are embedded as text, so their wording drives the result.
- The documents are non-empty strings.
- The account balance is positive and the caller uses a private API key.

## Decision Tree

1. How is a label chosen? The endpoint does not pick one: it returns every label ordered by similarity. Take the first `scores` entry for single-label routing, or apply a score threshold for multi-label routing.
2. Are the labels distinct? Overlapping or vague labels produce close scores. Rewrite them as descriptive phrases (for example, `billing question about an invoice` instead of `billing`).
3. Is the input large? Each document counts against the text-classification quota. Split large jobs into smaller requests or load `references/batch/`.
4. Does the task need reasoning, explanations, or free-form output? Classification is embedding similarity, not an LLM call. Use `references/decisions/` for question-based decisions.

## Construction

```json
{
  "documents": [
    "Document 1 text",
    "Document 2 text"
  ],
  "labels": ["billing", "support", "feature_request", "other"]
}
```

- Only `documents` and `labels` are read. There is no `model`, `language`, `temperature`, `max_labels`, or `metadata` field, and the embedding model is not selectable.

## Response Handling

1. `data.results` has one entry per document, in input order, with `index`, `document`, and `scores`.
2. `scores` lists every supplied label as `{ "label", "score" }`, ordered from highest to lowest cosine similarity. There is no `confidence` field.
3. Treat `scores` as relative: compare the top score with the next one, or calibrate a threshold on a labeled sample.
4. Route the document using the chosen label and record the top scores in the trace.

## Validation

- The top label matches the document at a glance. Spot-check a few.
- The label distribution is plausible. If 99% of documents land on one label, the labels may be too broad or too similar.
- The margin between the first and second score is meaningful for the routing rule you apply.

## Failure Modes

- `400`: a body is malformed, `documents` or `labels` is empty, or an entry is empty.
- `401` or `403`: the key is missing or invalid, a public key was used, or the balance is not positive.
- `429`: the text-classification quota was exceeded. Load `references/resilience/situations/rate-limit-429.md`.
- `503`: the embedding capacity is temporarily unavailable. Retry with backoff.
- Close scores or the wrong top label: the labels are ambiguous. Reword them as descriptive phrases.

## Escalation

- The classification is part of a larger pipeline: load `references/composition/situations/design-pipeline.md`.
- The task is large: load `references/batch/`.
- The classification needs to be evaluated: load `references/agentic-tests/situations/evaluate-by-criteria.md`.
- The task needs a reasoned or structured decision: load `references/decisions/`.

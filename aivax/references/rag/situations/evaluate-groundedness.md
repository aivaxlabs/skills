# Evaluate Groundedness

Use when the agent must measure whether a RAG answer is grounded in the retrieved evidence, or when the user wants to assess the overall quality of a collection's answers.

## Objective

Quantify the groundedness of the RAG answers and identify the failure mode (retrieval, ranking, generation) so the right fix can be applied.

## Preconditions

- A collection is available.
- A held-out set of questions with expected evidence is available (or can be built).
- The model and gateway are configured.

## Decision Tree

1. How many questions are in the held-out set? Small sets (10 to 30) are enough for a smoke test. Larger sets (100+) are needed for statistical confidence.
2. What is the source of the questions? Real user queries are best; synthetic questions from the documents are second-best.
3. What is the expected evidence? A document ID, a URL, or a textual fact. The expected evidence is the ground truth.
4. What is the metric? Precision (are the retrieved documents relevant?), recall (are all the relevant documents retrieved?), groundedness (does the answer cite the evidence?), faithfulness (does the answer stick to the evidence?).
5. How are the metrics computed? Manual review for small sets, an LLM-as-judge for larger sets, or a custom metric for specific cases.

## Construction

```text
1. Define the held-out set
   questions: [ { query: "...", expected_documents: ["..."], expected_answer: "..." } ]
   size: 30 to 100 questions
   diversity: cover the typical and the edge cases

2. Run the pipeline for each question
   for each question in questions:
     result = POST /api/v1/answer { terms: question.query, collections: [...] }
     record: retrieved_documents, cited_documents, answer

3. Compute the metrics
   precision: |retrieved ∩ expected| / |retrieved|
   recall: |retrieved ∩ expected| / |expected|
   groundedness: does the answer cite at least one expected document?
   faithfulness: does the answer stick to the cited evidence? (LLM-as-judge)

4. Report
   overall: precision, recall, groundedness, faithfulness
   per-question: the failing questions and their mode (retrieval, ranking, generation)
```

## Validation

- The metrics are computed on a held-out set, not on the training set.
- The failing questions are classified by mode.
- The report is reproducible: same questions, same metrics, same result.
- The trace ID is preserved.

## Limits

- LLM-as-judge is not the same as human review. Use it for volume, not for ground truth.
- A small sample can miss long-tail failures. Run a larger sample when the overall metrics are marginal.
- A high groundedness score with a low recall score means the model is sticking to the evidence but missing important evidence. Fix retrieval, not generation.

## Escalation

- The groundedness score is low: load `situations/debug-bad-answer.md`.
- The recall is low: load `situations/debug-missing-evidence.md`.
- The collection is consistently failing: load `situations/design-rag-pipeline.md` and consider redesigning.
- The user wants a built-in performance report: queue one with `POST /api/v1/collections/<id>/performances`. Confirm the cost with the user first.

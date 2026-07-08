# Evaluate RAG Quality

Use when the user wants to measure or improve retrieval quality across a collection.

Goal: use account-level evidence to assess whether collection retrieval is ready for production.

## Steps

1. Define evaluation questions and expected evidence.
2. Inspect collection state and document coverage.
3. Run direct retrieval for representative questions.
4. Review recent, low-quality, and high-quality RAG transactions.
5. Queue a built-in collection performance report only when the cost is justified:

```text
POST /api/v1/collections/<collection-id>/performances
```

6. Review findings, readiness, score, and remediation.
7. Recommend document, metadata, context, query strategy, or gateway tuning changes.

## Gotchas

- Performance reports can spend account balance.
- Good standalone retrieval does not prove gateway answers are good.
- Bad generation can look like bad retrieval unless the retrieved context is inspected.
- A small sample can miss long-tail failures.

## Validation

- Re-run the evaluation questions after changes.
- Inspect at least one real conversation using the collection.
- Report coverage gaps separately from ranking or generation gaps.


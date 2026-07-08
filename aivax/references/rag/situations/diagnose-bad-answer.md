# Diagnose Bad RAG Answer

Use when the user says an answer is wrong, incomplete, hallucinated, outdated, or missing citations.

Goal: determine whether the issue is retrieval, ranking, source data, gateway RAG settings, context construction, or generation.

## Steps

1. Identify the original user query, gateway, collection, time, and expected source.
2. View the conversation if the bad answer came from a real interaction.
3. Inspect linked gateway RAG settings.
4. Run direct retrieval:

```text
POST /api/v1/query
{
  "terms": ["<original or rewritten query>"],
  "collections": ["<collection-id>"],
  "top": 10,
  "minScore": 0.2,
  "reranker": "lexical",
  "includeReferences": true
}
```

5. Compare retrieved documents against the answer.
6. Search documents for the expected source.
7. Inspect recent or low-quality RAG transactions.
8. Classify the root cause and recommend the smallest fix.

## Common Causes

- Source document is not indexed.
- Correct document exists but chunk is weak or too noisy.
- Min score is too high.
- Top K is too low.
- Reranker demotes the correct result.
- Gateway is attached to the wrong collection.
- Query strategy rewrites the query poorly.
- Generation ignores the retrieved context.

## Validation

- Rerun the original query.
- Confirm the correct source appears in top results.
- Run a gateway test conversation.
- Confirm final answer uses or cites the expected source when citations are required.


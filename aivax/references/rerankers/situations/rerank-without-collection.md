# Rerank Without Collection

Use when the agent has a list of candidate document strings (from a custom search engine, a third-party API, or application state) and wants to re-order them with AIVAX without managing a collection.

## Objective

Get a relevance-ranked list of candidates without the cost of indexing, storing, and maintaining a managed AIVAX collection, while reusing the cache when the same query and document strings appear again.

## Preconditions

- The candidate documents are strings the application already owns.
- The query is known.
- The agent has decided that a managed collection is not justified (small corpus, dynamic candidates, no persistence requirement).

## Decision Tree

1. Is the candidate set dynamic or request-specific? Reflex is the right choice.
2. Is the candidate set large and stable? A managed collection is more cost-efficient. Compare token usage: sending 10,000 candidates on every request adds up.
3. Are the same query and document strings reused? Reflex caches by content hash. Reuse is the path to cache hits.
4. Is the language coverage a constraint? Verify with `aivax_list_models`.

## Construction

```text
POST /api/v1/generations/rerank
{
  "model": "@aivax/reflex-v1",
  "query": "<query>",
  "documents": [
    "<candidate 1>",
    "<candidate 2>",
    "..."
  ],
  "top_n": 5,
  "min_score": 0.2,
  "metadata": { "trace_id": "tr_..." }
}
```

Omit `model` to use Reflex by default. The endpoint accepts up to 10,000 documents and returns at most 200. The `top_n` and `min_score` are optional; `top_n` defaults to 5 (or the document count when fewer than 5 are supplied), `min_score` defaults to 0.

## Response Handling

1. The response is a list of results in descending relevance order.
2. Each result has the original zero-based `index`, the `relevance_score`, and the `document.text`.
3. The usage object reports `input_tokens`, `cached_input_tokens`, `total_tokens`, and `cost`.
4. The agent uses the result to feed the top-N to the next stage of the pipeline (typically a chat completion).

## Validation

- The response is a list in descending relevance order.
- The result's `index` matches the original document's position.
- The relevance score is between 0 and 1.
- The trace ID is preserved.
- The usage object is inspected to confirm cache hits when expected.

## Failure Modes

- The endpoint returns 400: the model is not autonomous, the document count exceeds the limit, or the input is empty. Inspect the error and the input.
- The endpoint returns 429: the reranking request quota or the Reflex token-rate quota is exhausted. Load `references/resilience/situations/rate-limit-429.md`.
- The result is poor: the candidates are noisy, the query is ambiguous, or the model is not strong on the domain. Switch the model or improve the candidates.

## Limits

- Reflex accepts up to 10,000 documents. Larger candidate sets must be split.
- The cache is account-scoped and is not permanent. Reuse within a window is the path to cache hits.
- The endpoint returns at most 200 results even when more documents are supplied.

## Escalation

- The candidate set is large and stable: load `references/rag/situations/design-rag-pipeline.md` and consider a managed collection.
- The cache hit rate is low: load `references/cost-monitoring/situations/optimize-spend.md` and consider whether a managed collection is more cost-efficient.
- The reranker is part of a larger pipeline: load `references/composition/situations/design-pipeline.md`.

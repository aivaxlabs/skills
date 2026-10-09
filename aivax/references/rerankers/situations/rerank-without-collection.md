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
2. Is the candidate set large and stable? Compare a managed collection's indexing and storage charges with repeated reranking and token usage.
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
  "min_score": 0.2
}
```

Omit `model` to use Reflex by default. Check the live catalog for document limits. The `top_n` and `min_score` are optional; consult the API contract for their defaults.

## Response Handling

1. Read the raw response object's `results` array, not a `data` envelope. Results are in descending relevance order.
2. Each result has the original zero-based `index`, the `relevance_score`, and the `document.text`.
3. For Reflex, `usage` reports `input_tokens`, `cached_input_tokens`, `total_tokens`, and `cost`. Other models can report different usage fields. Covered Reflex calls report no account charge and appear in subscription consumption, not billing history.
4. The agent uses the result to feed the top-N to the next stage of the pipeline (typically a chat completion).

## Validation

- The response's `results` array is in descending relevance order.
- The result's `index` matches the original document's position.
- The relevance score is between 0 and 1.
- The response `id` is recorded with the application's trace; request metadata is not echoed.
- The usage object is inspected to confirm cache hits when expected.

## Failure Modes

- The endpoint returns 400: the model is not autonomous, the document count exceeds the limit, or the input is empty. Inspect the error and the input.
- The endpoint returns 429: the reranking request quota or Reflex processing-time cap is reached. Load `references/resilience/situations/rate-limit-429.md`.
- The result is poor: the candidates are noisy, the query is ambiguous, or the model is not strong on the domain. Switch the model or improve the candidates.

## Limits

- Check the live catalog for candidate and result limits; split larger candidate sets as needed.
- The cache is account-scoped and is not permanent. Reuse within a window is the path to cache hits.

## Escalation

- The candidate set is large and stable: load `references/rag/situations/design-rag-pipeline.md` and consider a managed collection.
- The cache hit rate is low: load `references/cost-monitoring/situations/optimize-spend.md` and consider whether a managed collection is more cost-efficient.
- The reranker is part of a larger pipeline: load `references/composition/situations/design-pipeline.md`.

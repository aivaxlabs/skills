# Choose Reranker

Use when the agent must select a reranker for a task.

## Objective

Pick the cheapest reranker that meets the task's quality, latency, context, and document-count floors, with the right autonomy for the call site.

## Preconditions

- The candidate documents are available.
- The query is available.
- The call site is known: standalone `/api/v1/generations/rerank` requires `autonomousUse: true`; RAG settings can use any of `lexical`, `rrf`, `none`, or any autonomous model.

## Decision Tree

1. What is the total token count of the query plus the candidates? Pick a model whose context exceeds this. Segment if necessary.
2. How many candidate documents? Pick a model whose live catalog limit supports this count.
3. Is the language coverage a constraint? Check the live catalog and provider capabilities.
4. Is the volume high? Compare allowance-covered Reflex calls, metered cache hits, and other models; lexical has no usage charge but shares the reranking request quota.
5. Is precision the priority? Evaluate the candidate models on representative queries before choosing.
6. Is the call site RAG? `lexical`, `rrf`, `none`, or any autonomous model are accepted. The gateway's `rerankerName` parameter is the configuration point.
7. Is the call site standalone? Any autonomous model is accepted. RRF and `none` are rejected.

## Recommendation Output

- The chosen reranker name.
- A one-line rationale naming the binding constraint (cheapest with cache, longest context, multilingual, etc.).
- Estimated cost per call.
- Estimated latency class.
- The dominant alternative and when to switch to it.

## Validation

- The reranker is available on the user's plan.
- The reranker is `autonomousUse: true` if the call site is standalone.
- A small smoke test returns the expected order for a representative query and candidate set.
- The cost is within the user's cap.

## Limits

- The reranker re-orders the candidates. It cannot recover a document that is not in the candidates.
- The cost varies with the document count and the document length. Estimate before running.

## Escalation

- The candidates are not from a managed collection: load `situations/rerank-without-collection.md`.
- The reranker quota is exhausted: load `references/resilience/situations/rate-limit-429.md`.
- The reranker is part of a RAG pipeline: load `references/rag/`.

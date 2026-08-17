---
name: aivax-rerankers
description: Choose and apply a reranker to improve the order of retrieved candidate documents on AIVAX — Reflex, Cohere, Qwen, Jina, NVIDIA, or lexical. Load when the agent has a set of candidate documents that need to be re-ordered for a query, with or without a managed AIVAX collection.
---

# Rerankers

This sub-skill owns rerankers on AIVAX. A reranker takes a query and a list of candidate documents and returns the documents in a new order, with relevance scores. Rerankers do not search a collection; they re-order what the application already has.

## Operating Files

- `reranker-catalog.md`: the available rerankers, their limits, and their pricing.
- `situations/choose-reranker.md`: pick the right reranker for the task.
- `situations/rerank-without-collection.md`: rerank candidate documents without a managed AIVAX collection (Reflex standalone).

## Endpoint

`POST /api/v1/generations/rerank`. The endpoint accepts a query, a list of candidate document strings, a model name, an optional `top_n`, and an optional `min_score`. The response is a list of results in descending relevance order, each with the original zero-based index, the relevance score, and the document text.

The legacy `/api/v1/generations/reflex/rerank` route is an alias for the same endpoint. Use the generic route for new integrations.

## When To Use A Reranker

Use this sub-skill when the agent has a list of candidate documents (from semantic search, from a custom pipeline, or from a third-party search engine) and needs to re-order them by relevance to a query.

Do not use this sub-skill for:

- Searching a managed AIVAX collection. Use `references/rag/` instead.
- A direct chat completion. Use `references/text-inference/` instead.

## Reranker Selection

Rerankers differ in language coverage, context window, document limits, latency, and cost. The full catalog is in `reranker-catalog.md`. The short version:

- **Reflex** (`@aivax/reflex-v1`): default AIVAX reranker. Low latency, bounded lexical evidence, account-scoped cache. Best general default.
- **Qwen 3 Reranker** (`@qwen/qwen3-reranker-0.6b`, `4b`, `8b`): three sizes for different quality/latency trade-offs.
- **Jina Reranker v3** (`@jina/reranker-v3`): 131k token context, useful for very long documents.
- **Cohere Rerank 4** (`@cohere/rerank-4-pro`, `@cohere/rerank-4-fast`): search-unit pricing, large document support.
- **NVIDIA Nemotron** (`@nvidia/llama-nemotron-rerank-vl-1b-v2`): vision-language reranker, useful when the candidates include images.
- **Lexical**: word-aware reranking with no cost and no quota. Useful when the candidates are already high-quality and the reranker should be cheap.
- **RRF**: rank fusion for vector and lexical ranks. RAG only.

## Autonomy

The rerank endpoint accepts only models with `autonomousUse: true`. `rrf` and `none` are RAG-only and are not accepted here.

## Cost Awareness

Rerankers are priced per million tokens (most models) or per search unit (Cohere). Reflex has a separate cache-miss and cache-hit price. Reusing the same query or document strings can produce a cache hit, which is typically 5x cheaper. The usage object reports `input_tokens`, `cached_input_tokens`, `total_tokens`, and `cost`.

## Validation

- The response is a list of results in descending relevance order.
- The results preserve the original zero-based index of each document.
- The relevance score is between 0 and 1 (or the model's equivalent).
- The trace ID is preserved.

## Limits

- Each model has a declared context and document count limit. Reflex accepts up to 10,000 documents (returns at most 200). Most other models accept up to 1,024.
- The endpoint returns 400 for unknown or non-autonomous models, invalid `top_n` or `min_score`, empty input, or a document count above the model's limit.
- All non-none reranking operations share the account's reranking request quota. Reflex also has a token-rate quota. Exceeding either returns 429. Load `references/resilience/situations/rate-limit-429.md`.

## Escalation

- The candidates are not from a managed collection: load `situations/rerank-without-collection.md`.
- The reranker is the wrong choice: load `situations/choose-reranker.md` and try a different model.
- The reranker quota is exhausted: load `references/resilience/situations/rate-limit-429.md`.
- The reranker is part of a RAG pipeline: load `references/rag/`.

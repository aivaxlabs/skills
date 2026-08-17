# Reranker Catalog

The current AIVAX reranking catalog. The list is a moving target; treat the live `aivax_list_models` or `GET /api/v1/information/rerankers` as the source of truth. This file is the shape and the selection criteria.

## Catalog Snapshot

| Name | Autonomous | Context | Documents | Pricing |
| --- | --- | --- | --- | --- |
| `@aivax/reflex-v1` | Yes | 2,048 tokens | 10,000 | $0.015 / mtok cache miss, $0.003 / mtok cache hit |
| `@jina/reranker-v3` | Yes | 131,072 tokens | Catalog-declared | $0.05 / mtok |
| `@qwen/qwen3-reranker-0.6b` | Yes | 32,768 tokens | 1,024 | $0.01 / mtok |
| `@qwen/qwen3-reranker-4b` | Yes | 32,768 tokens | 1,024 | $0.025 / mtok |
| `@qwen/qwen3-reranker-8b` | Yes | 32,768 tokens | 1,024 | $0.05 / mtok |
| `@nvidia/llama-nemotron-rerank-vl-1b-v2` | Yes | 10,240 tokens | 1,024 | $0.01 / mtok |
| `@cohere/rerank-4-pro` | Yes | 32,768 tokens | 10,000 | $0.0025 / search unit |
| `@cohere/rerank-4-fast` | Yes | 32,768 tokens | 10,000 | $0.002 / search unit |
| `@cohere/rerank-3.5` | Yes | 4,096 tokens | 10,000 | $0.001 / search unit |
| `lexical` | Yes | No catalog limit | No catalog limit | No cost |
| `rrf` | No | RAG only | RAG only | No cost |
| `none` | No | RAG only | RAG only | No cost |

`smart` is an accepted compatibility alias for `@aivax/reflex-v1`, but it is not a separate catalog model. Prefer canonical model names in stored configuration and new integrations.

## Field Meanings

- **Autonomous**: whether the model can be used in the standalone `/api/v1/generations/rerank` endpoint. `rrf` and `none` are RAG-only.
- **Context**: the maximum total tokens (query + documents) the model can process in a single call. Documents longer than the context are rejected; segment them first with `references/text-tools/situations/segment-text.md`.
- **Documents**: the maximum number of candidate documents the model accepts in a single call. Reflex accepts up to 10,000 but returns at most 200 results; most others accept up to 1,024.
- **Pricing**: per million input tokens (mtok) for token-priced models, or per search unit for Cohere. Reflex distinguishes cache miss from cache hit.

## Selection By Use Case

| Use case | Recommended reranker | Why |
| --- | --- | --- |
| Default general use | `@aivax/reflex-v1` | Low latency, account-scoped cache, good quality. |
| Very long documents | `@jina/reranker-v3` | 131k token context. |
| Multilingual retrieval | `@qwen/qwen3-reranker-4b` or `8b` | Strong multilingual coverage. |
| Vision-language candidates | `@nvidia/llama-nemotron-rerank-vl-1b-v2` | Accepts image-based candidates. |
| High-volume, low-cost | `lexical` | No cost, no quota, word-aware. |
| High-volume with semantic quality | `@aivax/reflex-v1` (cache) | Cache hit is 5x cheaper than miss. |
| High precision, low volume | `@cohere/rerank-4-pro` | High quality, search-unit pricing. |
| High speed, lower precision | `@cohere/rerank-4-fast` or `@qwen/qwen3-reranker-0.6b` | Smaller models, faster output. |
| RAG-only rank fusion | `rrf` | Combines vector and lexical ranks. |
| RAG without reranking | `none` | Disables reranking entirely. |

## Caveats

- A reranker re-orders what it is given. If the relevant document is not in the candidates, no reranker will help. Fix retrieval, not the reranker.
- The same `id` can mean different things on different plans. Verify availability before recommending.
- A model's pricing can change. Verify with `aivax_list_models` or the public pricing page.
- The document count limit is per call. A request that sends 10,000 documents to a model with a 1,024 limit returns 400.

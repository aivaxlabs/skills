# Reranker Catalog

The current AIVAX reranking catalog. The list is a moving target; treat the live `aivax_list_models` or `GET /api/v1/information/rerankers-models.json` as the source of truth. This file is the shape and the selection criteria.

## Catalog Lookup

Use `GET /api/v1/information/rerankers-models.json` to inspect current names, `autonomousUse`, `technicalInformation`, `pricingDescription`, and `subscriptionUsage` rather than copying a fixed catalog. Reflex is allowance-eligible; other paid rerankers are metered. Lexical, RRF, and none have no usage charge. `smart` is a compatibility alias for Reflex, not a separate catalog model; prefer the canonical name returned by the catalog.

## Field Meanings

- **Autonomous**: whether the model can be used in the standalone `/api/v1/generations/rerank` endpoint. `rrf` and `none` are RAG-only.
- **Context**: the maximum total tokens (query + documents) the model can process in a single call. Documents longer than the context are rejected; segment them first with `references/text-tools/situations/segment-text.md`.
- **Documents**: the maximum number of candidate documents the model accepts in a single call, when declared. The result limit may be smaller.
- **Pricing**: token rates or search-unit rates for metered calls. Reflex first uses its separate daily allowance when the entire call fits; otherwise cache-miss and cache-hit rates apply. Cached and uncached input both count toward the allowance. Covered calls appear in subscription consumption, not billing history.

## Selection By Use Case

| Use case | Recommended reranker | Why |
| --- | --- | --- |
| Default general use | `@aivax/reflex-v1` | Low latency, account-scoped cache, good quality. |
| Very long documents | Jina | Check the current context limit. |
| Multilingual retrieval | Qwen | Verify language coverage in the live catalog. |
| Vision-language candidates | `@nvidia/llama-nemotron-rerank-vl-1b-v2` | Accepts image-based candidates. |
| High-volume, low-cost | `lexical` | No usage charge; reranking request quota still applies. |
| High-volume with semantic quality | Reflex (allowance/cache) | Covered calls incur no charge; metered cache hits cost less than misses. |
| High precision, low volume | `@cohere/rerank-4-pro` | High quality, search-unit pricing. |
| High speed, lower precision | `@cohere/rerank-4-fast` or `@qwen/qwen3-reranker-0.6b` | Smaller models, faster output. |
| RAG-only rank fusion | `rrf` | Combines vector and lexical ranks. |
| RAG without reranking | `none` | Disables reranking entirely. |

## Caveats

- A reranker re-orders what it is given. If the relevant document is not in the candidates, no reranker will help. Fix retrieval, not the reranker.
- Verify model availability on the account's plan before recommending it.
- A model's pricing can change. Verify with `aivax_list_models` or the public pricing page.
- The document count limit is per call. Exceeding the model's declared limit is rejected.

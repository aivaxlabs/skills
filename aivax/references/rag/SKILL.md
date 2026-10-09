---
name: aivax-rag
description: Build and operate Retrieval-Augmented Generation on AIVAX — collections, ingestion, semantic search, answer generation, chunking, and grounding evaluation. Load when the agent must retrieve documents for a query, generate a grounded answer, or design a RAG pipeline.
---

# RAG

This sub-skill owns retrieval-augmented generation on AIVAX: collections, documents, semantic search, answer generation, and grounding evaluation. The endpoint surface is `POST /api/v1/query` (semantic search) and `POST /api/v1/answer` (answer generation). The collections and documents are managed through the account API.

## Operating Files

- `situations/design-rag-pipeline.md`: design a RAG pipeline from scratch.
- `situations/debug-missing-evidence.md`: the expected document is not in the retrieval results.
- `situations/debug-bad-answer.md`: the answer is wrong, incomplete, hallucinated, or uncited.
- `situations/evaluate-groundedness.md`: measure whether the answer is grounded in the retrieved evidence.

## When To Use RAG

Use this sub-skill when the agent must:

- Retrieve documents from one or more collections for a query.
- Generate a grounded answer that cites the retrieved evidence.
- Design or debug a RAG pipeline.
- Evaluate the quality of a collection (coverage, retrieval, grounding).
- Operate the gateway RAG settings (top K, min score, reranker, query strategy).

Do not use this sub-skill for:

- Reranking candidates without a collection (load `references/rerankers/situations/rerank-without-collection.md`).
- A direct chat completion without retrieval (load `references/text-inference/`).
- A classification or segmentation task (load `references/text-tools/`).

## Core Concepts

- **Collection**: a semantic knowledge library that stores documents and indexes them with embeddings.
- **Document**: a single piece of collection knowledge, such as an article chunk, a manual section, a product description, or another RAG item.
- **Embedding**: a vector representation of the document, computed at indexing time. Semantic queries embed the search terms; filter-only requests do not.
- **Semantic search**: the process of finding the most relevant documents for a query, by embedding similarity and (optionally) a reranker.
- **Answer generation**: the process of generating a grounded answer from the retrieved documents. The answer cites the evidence.
- **Query strategy**: how the query is transformed before embedding. Strategies include `Plain`, `Concatenate`, `FullRewrite`, `UserRewrite`, and `QueryFunction`.

## Endpoints

- `POST /api/v1/query`: semantic search or filter-only retrieval. Returns documents with scores in `data`.
- `POST /api/v1/answer`: answer generation. Returns a generated answer that cites the evidence.
- `GET /api/v1/collections`: list collections.
- `POST /api/v1/collections`: create an empty collection from `collectionName` with optional `context` and `tags`.
- `GET /api/v1/collections/<id>`: view a collection.
- `GET /api/v1/collections/<id>/export.jsonl`: export all documents as JSONL.
- `GET /api/v1/collections/<id>/documents`: browse documents.
- `GET /api/v1/collections/<id>/documents/<document-id>`: view one document.
- `GET /api/v1/collections/<id>/transactions`: list recent, low-quality, or high-quality transactions.
- `GET /api/v1/collections/<id>/transactions/<transaction-id>`: view one transaction in detail.
- `GET /api/v1/collections/<id>/transactions/export/jsonl?view=<mode>`: export the selected transaction view as JSONL.
- `GET /api/v1/collections/<id>/deduplication/estimate?delta=<level>`: estimate exact and semantic duplicate removal; `delta` accepts `low`, `medium`, or `high` and defaults to `medium`.
- `POST /api/v1/collections/<id>/deduplication?delta=<level>`: start the corresponding duplicate removal (destructive; require explicit approval).
- `PUT /api/v1/collections/<id>`: edit collection metadata.
- `PUT /api/v1/collections/<id>/documents`: upsert one document.
- `POST /api/v1/collections/<id>/documents`: import JSONL through the multipart file field `documents` (route name: `system.v1.documents.index-jsonl`, not a URL suffix). The optional `insert-mode=sync` also deletes documents absent from the upload; use only with explicit authorization.
- `POST /api/v1/collections/<id>/documents/media/chunk`: append a binary chunk identified by the required media-upload headers.
- `POST /api/v1/collections/<id>/documents/media/complete`: queue processing of the uploaded media chunks.
- `DELETE /api/v1/collections/<id>/documents/<document-id>`: delete one document; `include-references` also deletes documents sharing its reference (destructive; require explicit approval).
- `DELETE /api/v1/collections/<id>`: delete the collection and its documents (destructive; require explicit approval).
- `DELETE /api/v1/collections/<id>/reset-only`: delete every document while retaining the collection configuration (destructive; require explicit approval).
- `DELETE /api/v1/collections/<id>/vectors-only`: delete and rebuild the vector index while preserving documents (destructive mutation; require explicit approval).

## Choose The Retrieval Mode

- Send `collections` and use `terms` as the canonical search field on both `/query` and `/answer`. `terms` accepts a string or an array of strings; blank terms are discarded.
- Terms only: semantic search. Terms plus `filter`: restrict eligible documents before semantic ranking. Filter only: return matching indexed documents newest first, up to `top`, with `score: 1`; no query embedding or reranking runs. This score is not relevance, and `minScore` does not narrow filter-only results.
- Use `/query` for filter-only lookups. `/answer` still generates an answer after retrieval, so supply a meaningful question when using it.
- Neither usable terms nor a filter returns `400`. Filters do not bypass indexing: queued documents are not returned.

## Apply Exact Constraints

- Use the singular `filter`, never `filters` (which is ignored). HTTP requests accept a filter string or an array of strings combined with `and`. Omitted, `null`, or an empty array means no filter; an empty string is invalid.
- Filter fields are `name`, `content`, `tags`, `createdAt`, `updatedAt`, and `metadata.<key>`. Use `tags has "policy"`, `tags in ("faq", "policy")`, or `metadata.region = "latam"`; combine with `and`, `or`, `not`, and parentheses. Use terms for meaning, filters for exact constraints.
- Text comparisons ignore case and accents. Metadata comparisons preserve value types: a number does not match a numeric string. Prefer an explicit offset or `Z` in date conditions. Consult [Document Filters](https://docs.aivax.net/docs/filters/document-filters) for operators, date syntax, and validation limits rather than guessing.
- Invalid types, syntax, fields, operators, or filter-limit violations return `400`; parse errors identify the problem and, for arrays with multiple expressions, the failing item index. A filter evaluation timeout also returns `400`: make the conditions more selective rather than retrying unchanged. No matching documents is a successful empty result, not an error, and skips query embedding and reranking.
- Collections MCP search and the gateway `query` tool accept an optional filter string. Automatic gateway retrieval does not apply filters; do not invent a gateway-level filter setting.
- Filters constrain matched documents, not authorization. With `includeReferences: true`, `referencedDocuments` are expanded separately without reapplying the filter. Keep it false when all returned content must satisfy the constraint.
- The document-browsing endpoint's query-string `filter` uses a different search syntax; do not copy RAG filter expressions into it.

## Minimal Examples

```json
POST /api/v1/query
{
  "collections": ["<collection-id>"],
  "terms": "how do refunds work?",
  "filter": "tags has \"policy\"",
  "top": 5
}
```

```json
POST /api/v1/answer
{
  "collections": ["<collection-id>"],
  "prompt": "how do refunds work?"
}
```

Split long sources into retrieval-ready documents before upsert or JSONL import; ordinary document ingestion does not segment them automatically. Use `references/text-tools/situations/segment-text.md` when segmentation is needed.

## Media Injector

Use Media Injector when the useful knowledge sits inside a source file (PDF report, manual, policy, image, scan, audio, video) that has not already been split into retrieval-ready text. Upload the file through the collection's Import from files flow (`media/chunk` then `media/complete`); each file spawns a media-processing job tracked under Batch, with states `queued`, `processing`, `completed`, `failed`, or `cancelled`. The job writes concise factual documents in the source's predominant language and queues them for indexing; it does not preserve the source as one searchable document. Optional processing context steers which facts matter but cannot add absent facts. After completion, review the generated documents (wording, coverage, sensitive data) and wait for indexing before testing search. Prefer direct document upsert or JSONL import when exact wording, deterministic boundaries, stable names, or application-controlled metadata are required.

## Validation

- Inspect `/query` results in `data`, or `/answer` results in `data.queryResults` alongside `data.answer`.
- Matched collection documents include `timestamps.createdAt` and `timestamps.updatedAt` as date-time strings in `yyyy-MM-ddTHH:mm:ss` form, without a timezone suffix. Do not read them as Unix timestamps or silently assume UTC; referenced-document entries do not carry these fields.
- The retrieved documents match the expected evidence and filter. Filter-only results prove eligibility, not semantic relevance.
- The generated answer cites the retrieved documents.
- The grounding score is acceptable. See `situations/evaluate-groundedness.md`.
- The trace ID is preserved.

## Limits

- Per-collection and per-account quotas exist. Plan limits control collection count, search rate, insertion rate, and JSONL import size.
- Search and insertion embeddings share the plan's daily RAG embedding allowance. Each uncached embedding item is either fully covered or billed normally; cached query embeddings do not consume it. Included items appear in subscription consumption, not billing history.
- Reflex reranking uses a separate allowance; answer generation and other rerankers are billed separately. Allowances do not bypass balance requirements or rate limits.
- Some query strategies are more expensive than others. `FullRewrite` and `UserRewrite` consume tokens; `Plain` does not.

## Escalation

- The expected document is missing: load `situations/debug-missing-evidence.md`.
- The answer is wrong: load `situations/debug-bad-answer.md`.
- The user wants a custom reranker: load `references/rerankers/`.
- The task is to design a new pipeline: load `situations/design-rag-pipeline.md`.
- The request fails with 429 or a transient error: load `references/resilience/situations/rate-limit-429.md`.

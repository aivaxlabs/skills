---
name: aivax-rag
description: Build and operate Retrieval-Augmented Generation on AIVAX — collections, ingestion, semantic search, answer generation, chunking, and grounding evaluation. Load when the agent must retrieve documents for a query, generate a grounded answer, or design a RAG pipeline.
---

# RAG

This sub-skill owns retrieval-augmented generation on AIVAX: collections, documents, semantic search, answer generation, and grounding evaluation. The endpoint surface is `POST /api/v2/query` (semantic search), `POST /api/v2/query/similar` and `POST /api/v2/query/browse` (similar documents and listing), and `POST /api/v1/answer` (answer generation). `POST /api/v1/query` is deprecated but still served. The collections and documents are managed through the account API.

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

- **Collection**: a semantic knowledge library that stores documents and indexes them with embeddings. Names starting with `@` are reserved for system collections (for example `@memories`, which holds long-term user memories; see `references/memories/`): creating or renaming to such a name, or editing or generating context for one, returns 403. Do not use them for application knowledge.
- **Embedder**: one platform-wide embedder indexes and queries every collection; it is not selectable per collection or request, and it is not part of the model catalog. Do not look for an embedding model setting. To read the current embedder name, context size, and recommended chunk size (use it to size chunks), call `GET /api/v1/information/collection-embedding-model.json`; it needs no authentication and the embedder can change over time.
- **Document**: a single piece of collection knowledge, such as an article chunk, a manual section, a product description, or another RAG item.
- **Embedding**: a vector representation of the document, computed at indexing time. Semantic queries embed the search terms; filter-only requests do not.
- **Semantic search**: the process of finding the most relevant documents for a query, by embedding similarity and (optionally) a reranker.
- **Answer generation**: the process of generating a grounded answer from the retrieved documents. The answer cites the evidence.
- **Query strategy**: how the query is transformed before embedding. Strategies include `Plain`, `Concatenate`, `FullRewrite`, `UserRewrite`, and `QueryFunction`.

## Endpoints

- `POST /api/v2/query`: semantic search with optional reranking and cursor pages. See Query v2 below.
- `POST /api/v2/query/similar`: documents closest to one indexed document, from its stored vector.
- `POST /api/v2/query/browse`: list indexed documents in a fixed order, optionally filtered, without semantic search.
- `POST /api/v1/query`: deprecated semantic search or filter-only retrieval. Returns documents with scores in `data`.
- `POST /api/v1/answer`: answer generation. Returns a generated answer that cites the evidence.
- `GET /api/v1/collections`: list collections.
- `POST /api/v1/collections`: create an empty collection from `collectionName` with optional `context` and `tags`.
- `GET /api/v1/collections/<id>`: view a collection.
- `GET /api/v1/collections/<id>/export.jsonl`: export all documents as JSONL.
- `GET /api/v1/collections/<id>/documents`: browse documents (query `filter`, `order_by`, `state`). Items carry `id`, `createdAt`, `updatedAt`, `name`, `reference`, `tags`, `contentsPreview`, `indexState`, and `approximateTokenCount` (an estimate from text length, useful for spotting documents that need segmentation).
- `GET /api/v1/collections/<id>/documents/<document-id>`: view one document.
- `GET /api/v1/collections/<id>/transactions`: list recent, low-quality, or high-quality transactions. Transactions are recorded only while the account's `allowRagLogging` is on (`references/account/`).
- `GET /api/v1/collections/<id>/transactions/<transaction-id>`: view one transaction in detail.
- `GET /api/v1/collections/<id>/transactions/export/jsonl?view=<mode>`: export the selected transaction view as JSONL.
- `GET /api/v1/collections/<id>/deduplication/estimate?delta=<level>`: estimate exact and semantic duplicate removal; `delta` accepts `low`, `medium`, or `high` and defaults to `medium`.
- `POST /api/v1/collections/<id>/deduplication?delta=<level>`: start the corresponding duplicate removal (destructive; require explicit approval).
- `PUT /api/v1/collections/<id>`: edit collection metadata.
- `PUT /api/v1/collections/<id>/documents`: upsert one document.
- `GET /api/v2/collections/<id>/documents/index-jsonl` (WebSocket upgrade): preferred JSONL import for new integrations. Send text messages with complete JSONL lines (`docId` and `text` required; optional `__ref`, `__tags`, `__meta`) and finish with a line containing only `END`. The server replies with JSON `progress` (`committed`, `updated`, `skipped`), `completed` (`received`, `updated`, `skipped`, `removed`), or `error` (`message`, `line`, `committed`) messages. Lines are committed and queued in blocks, so after a failure or disconnect before `END`, resend only the lines after the reported `committed` count. Only one import per collection runs at a time (409 when another does not finish in time), and a long pause between messages closes the connection. Browsers can authenticate with the `api-key` query parameter. `insert-mode=sync` applies as below.
- `POST /api/v1/collections/<id>/documents`: deprecated multipart JSONL import through the file field `documents` (route name: `system.v1.documents.index-jsonl`, not a URL suffix); it accepts a smaller per-request line limit than the WebSocket import (see Plans and Limits). The optional `insert-mode=sync` (both imports) also deletes documents absent from the upload; use only with explicit authorization.
- `POST /api/v1/collections/<id>/documents/media/chunk`: append a binary chunk identified by the required media-upload headers.
- `POST /api/v1/collections/<id>/documents/media/complete`: queue processing of the uploaded media chunks.
- `DELETE /api/v1/collections/<id>/documents/<document-id>`: delete one document; `include-references` also deletes documents sharing its reference (destructive; require explicit approval).
- `DELETE /api/v1/collections/<id>`: delete the collection and its documents (destructive; require explicit approval).
- `DELETE /api/v1/collections/<id>/reset-only`: delete every document while retaining the collection configuration (destructive; require explicit approval).
- `DELETE /api/v1/collections/<id>/vectors-only`: clear the vectors of documents that are not indexed or were embedded by a different embedder, and re-queue them for indexing; documents and already-current vectors are preserved. Returns how many documents were enqueued (mutation; require explicit approval). It does not force a rebuild of healthy vectors.

## Query v2

- Shared fields: `collections` (IDs, or `{ "id", "filter" }` objects whose filter is combined with the top-level `filter`; listing a collection twice is `400`), `filter` (string or array combined with `and`), `sort` (`{ "by": "score" | "createdAt" | "updatedAt" | "indexedAt" | "name", "dir": "asc" | "desc" }`), `limit` (1 to 128, default 10), `cursor`, and `include` (`content`, `metadata`, `references`; default content and metadata).
- `/api/v2/query`: `terms` is required; each document is scored by its best match among the terms. `minScore` is 0 to 0.99 (default 0.2). `candidates` is the pool that is reranked, sorted, and paged: from `limit` to 1000, default `limit × 3`, and at most the reranker's document limit. `reranker` is `smart` (Reflex, the default), `lexical`, `rrf`, `none`, or a full reranker name. An explicit reranker with a sort other than score is `400`; without one, a non-score sort disables reranking.
- `/api/v2/query/similar`: `documentId` (an indexed document of the account) is required; `collections` defaults to the source document's collection. Nothing is embedded or reranked, so there is no embedding or reranking cost, and the source document is never returned.
- `/api/v2/query/browse`: `collections` is required and no terms are read. Only indexed documents are listed; `sort` defaults to `createdAt` ascending, and sorting by score is `400`. A filter that takes too long is `400`; make it more selective.
- Response `data`: `results[]` with `rank`, `document` (`id`, `collectionId`, `name`, `reference`, `tags`, `createdAt`, `updatedAt`, `indexedAt`, plus `content` and `metadata` when included), `scores` (`similarity`, `rerank`, `final`, which is the rerank score when a reranker ran, otherwise the similarity), and `references[]` (`id`, `content`) when requested; `page` (`cursor`, `hasMore`, `candidates`); and `reranker`. Browse results have no `rank` or `scores`, and their `page` has no `candidates`; browse responses have no `reranker`.
- Paging: resend the same body with `page.cursor`; a different body is `400`. Query and similar pages reuse the pool without new cost, and their cursors expire after 10 minutes without use (`410`: repeat without a cursor). Browse cursors do not expire.
- All three accept public keys, require a positive balance, and count against the plan's search rate limit; reranked queries also count against the reranking limit.

## Choose The Retrieval Mode

These rules describe the deprecated `/api/v1/query` and `/api/v1/answer`.

- Send `collections` and use `terms` as the canonical search field on both `/query` and `/answer`. `terms` accepts a string or an array of strings; blank terms are discarded.
- Terms only: semantic search. Terms plus `filter`: restrict eligible documents before semantic ranking. Filter only: return matching indexed documents newest first, up to `top`, with `score: 1`; no query embedding or reranking runs. This score is not relevance, and `minScore` does not narrow filter-only results.
- Use `/query` for filter-only lookups. `/answer` still generates an answer after retrieval, so supply a meaningful question when using it.
- Neither usable terms nor a filter returns `400`. Filters do not bypass indexing: queued documents are not returned.

## Apply Exact Constraints

- Use the singular `filter`, never `filters` (which is ignored). HTTP requests accept a filter string or an array of strings combined with `and`. Omitted, `null`, or an empty array means no filter; an empty string is invalid.
- Filter fields are `name`, `content`, `reference`, `tags`, `createdAt`, `updatedAt`, and `metadata.<key>`. Use `tags has "policy"`, `tags in ("faq", "policy")`, or `metadata.region = "latam"`; combine with `and`, `or`, `not`, and parentheses. Use terms for meaning, filters for exact constraints.
- Text comparisons ignore case and accents. Metadata comparisons preserve value types: a number does not match a numeric string. Prefer an explicit offset or `Z` in date conditions. Consult [Document Filters](https://docs.aivax.net/docs/filters/document-filters) for operators, date syntax, and validation limits rather than guessing.
- Invalid types, syntax, fields, operators, or filter-limit violations return `400`; parse errors identify the problem and, for arrays with multiple expressions, the failing item index. A filter evaluation timeout also returns `400`: make the conditions more selective rather than retrying unchanged. No matching documents is a successful empty result, not an error, and skips query embedding and reranking.
- Collections MCP search and the gateway `query` tool accept an optional filter string. Automatic gateway retrieval does not apply filters; do not invent a gateway-level filter setting.
- Filters constrain matched documents, not authorization. With `includeReferences: true`, `referencedDocuments` are expanded separately without reapplying the filter. Keep it false when all returned content must satisfy the constraint.
- The document-browsing endpoint's query-string `filter` uses a different search syntax; do not copy RAG filter expressions into it. `/api/v2/query/browse` takes RAG filter expressions.

## Collections MCP

The Streamable HTTP server at `https://inference.aivax.net/v1/mcp/collections` exposes collections as tools, authenticated with an account bearer key.

- Configure it with headers: `X-Mcp-Collection-Id` (required, comma-separated collection IDs), `X-Mcp-Collection-Name` (tool name prefix), `X-Mcp-Naming-Convention` (`default` or `agent`), `X-Mcp-Top-K`, `X-Mcp-Min-Score`, `X-Mcp-Reranker`, and `X-Mcp-Allow-Write: yes` for the write and delete tools.
- Tools, as `default` / `agent` names after the prefix: search (`_search` / `_read`), browse (`_browse_documents` / `_list`), get document (`_get_document` / `_get`), write (`_create_or_update_document` / `_write`), and delete (`_delete_document` / `_delete`).
- Browse lists indexed documents without semantic search: optional `filter`, `sort` (`createdAt` default, `updatedAt`, `indexedAt`, `name`), `descending`, `include_content` (default false), `limit` (1 to 50, default 20), and `cursor` (the returned `next_cursor`, sent with the same sort, descending, and filter).
- Get document reads one document by its exact `name`, including documents not yet indexed, with optional 1-based `start_line` and `end_line`. One call returns at most 100,000 characters; read long documents in line ranges.
- Select tools with `X-Mcp-Enabled-Tools: search, browse` (keys `search`, `browse`, `write`, `delete`; `browse` also exposes get document; write and delete still need `X-Mcp-Allow-Write: yes`). Omitting the header exposes all tools; an empty or unrecognized allowlist exposes none. Refresh MCP discovery after changing it.
- Search, browse, and get document count against the plan's search rate limit.

## Minimal Examples

```json
POST /api/v2/query
{
  "collections": ["<collection-id>"],
  "terms": "how do refunds work?",
  "filter": "tags has \"policy\"",
  "limit": 5
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

- Inspect `/api/v2/query` results in `data.results`, `/api/v1/query` results in `data`, or `/answer` results in `data.queryResults` alongside `data.answer`.
- Matched collection documents include `timestamps.createdAt` and `timestamps.updatedAt` as date-time strings in `yyyy-MM-ddTHH:mm:ss` form, without a timezone suffix. Do not read them as Unix timestamps or silently assume UTC; referenced-document entries do not carry these fields.
- Imported or upserted documents are searchable only after `indexState` is `indexed`; confirm with the browse `state` filter before testing retrieval.
- The retrieved documents match the expected evidence and filter. Filter-only results prove eligibility, not semantic relevance.
- The generated answer cites the retrieved documents.
- The grounding score is acceptable. See `situations/evaluate-groundedness.md`.
- The trace ID is preserved.

## Limits

- Per-collection and per-account quotas exist. Plan limits control collection count, search rate, insertion rate, and JSONL import size.
- Search and insertion embeddings share the plan's daily RAG embedding allowance. Each uncached embedding item is either fully covered or billed normally; cached query embeddings do not consume it. Included items appear in subscription consumption, not billing history. Billed embeddings are charged from the cost actually reported for that embedding call, not from a catalog price, so use the Pricing page and the account's usage records rather than estimating from a model list.
- Indexing is asynchronous and shared across accounts. Queued documents are drained in batches that interleave collections, so a large import in one collection does not starve others but a backlog still delays searchability. A failed embedding batch stays `queued` and is retried by the next run; the document is not lost and not marked as an error. If documents stay `queued` for long, wait and recheck before re-uploading; re-uploading unchanged text does not speed indexing.
- Reflex reranking uses a separate allowance; answer generation and other rerankers are billed separately. Allowances do not bypass balance requirements or rate limits.
- Some query strategies are more expensive than others. `FullRewrite` and `UserRewrite` consume tokens; `Plain` does not.

## Escalation

- The expected document is missing: load `situations/debug-missing-evidence.md`.
- The answer is wrong: load `situations/debug-bad-answer.md`.
- The user wants a custom reranker: load `references/rerankers/`.
- The task is to design a new pipeline: load `situations/design-rag-pipeline.md`.
- The request fails with 429 or a transient error: load `references/resilience/situations/rate-limit-429.md`.

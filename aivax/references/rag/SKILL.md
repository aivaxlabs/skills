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

- **Collection**: a semantic knowledge library that stores documents and indexes them with embeddings. A collection can hold tens of thousands of documents.
- **Document**: a single piece of collection knowledge, such as an article chunk, a manual section, a product description, or another RAG item.
- **Embedding**: a vector representation of the document, computed at indexing time. The query is embedded in query mode at retrieval time.
- **Semantic search**: the process of finding the most relevant documents for a query, by embedding similarity and (optionally) a reranker.
- **Answer generation**: the process of generating a grounded answer from the retrieved documents. The answer cites the evidence.
- **Query strategy**: how the query is transformed before embedding. Strategies include `Plain`, `Concatenate`, `FullRewrite`, `UserRewrite`, and `QueryFunction`.

## Endpoints

- `POST /api/v1/query`: semantic search. Returns a list of documents with scores.
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

## Minimal Examples

```json
POST /api/v1/query
{
  "collections": ["<collection-id>"],
  "terms": "how do refunds work?",
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

Chunking is controlled at ingestion time; bulk document loads use the JSONL documents import (route name: `system.v1.documents.index-jsonl`).

## Media Injector

Use Media Injector when the useful knowledge sits inside a source file (PDF report, manual, policy, image, scan, audio, video) that has not already been split into retrieval-ready text. Upload the file through the collection's Import from files flow (`media/chunk` then `media/complete`); each file spawns a media-processing job tracked under Batch, with states `queued`, `processing`, `completed`, `failed`, or `cancelled`. The job writes concise factual documents in the source's predominant language and queues them for indexing; it does not preserve the source as one searchable document. Optional processing context steers which facts matter but cannot add absent facts. After completion, review the generated documents (wording, coverage, sensitive data) and wait for indexing before testing search. Prefer direct document upsert or JSONL import when exact wording, deterministic boundaries, stable names, or application-controlled metadata are required.

## Validation

- The retrieved documents match the expected evidence.
- The generated answer cites the retrieved documents.
- The grounding score is acceptable. See `situations/evaluate-groundedness.md`.
- The trace ID is preserved.

## Limits

- Per-collection and per-account quotas exist. Plan limits control collection count, search rate, insertion rate, and JSONL import size.
- Some query strategies are more expensive than others. `FullRewrite` and `UserRewrite` consume tokens; `Plain` does not.

## Escalation

- The expected document is missing: load `situations/debug-missing-evidence.md`.
- The answer is wrong: load `situations/debug-bad-answer.md`.
- The user wants a custom reranker: load `references/rerankers/`.
- The task is to design a new pipeline: load `situations/design-rag-pipeline.md`.
- The request fails with 429 or a transient error: load `references/resilience/situations/rate-limit-429.md`.

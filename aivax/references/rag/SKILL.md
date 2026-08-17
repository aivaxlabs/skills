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
- `GET /api/v1/collections/<id>`: view a collection.
- `GET /api/v1/collections/<id>/documents`: browse documents.
- `GET /api/v1/collections/<id>/transactions`: list recent, low-quality, or high-quality transactions.
- `POST /api/v1/collections/<id>/performances`: queue a performance report.
- `PUT /api/v1/collections/<id>`: edit collection metadata.
- `PUT /api/v1/collections/<id>/documents`: upsert one document.
- `POST /api/v1/collections/<id>/documents`: import many documents as JSONL.
- `DELETE /api/v1/collections/<id>/vectors-only`: rebuild vectors without deleting documents.

## Cost Awareness

A performance report can spend account balance. Queue it intentionally, then review `OverallScore`, `OverallReadiness`, findings, and remediation plan before editing documents.

## Validation

- The retrieved documents match the expected evidence.
- The generated answer cites the retrieved documents.
- The grounding score is acceptable. See `situations/evaluate-groundedness.md`.
- The trace ID is preserved.

## Limits

- Per-collection and per-account quotas exist. Plan limits control collection count, search rate, insertion rate, and JSONL import size.
- Some query strategies are more expensive than others. `FullRewrite` and `UserRewrite` consume tokens; `Plain` does not.
- Performance reports are not free. Confirm the cost with the user before queueing.

## Escalation

- The expected document is missing: load `situations/debug-missing-evidence.md`.
- The answer is wrong: load `situations/debug-bad-answer.md`.
- The user wants a custom reranker: load `references/rerankers/`.
- The task is to design a new pipeline: load `situations/design-rag-pipeline.md`.

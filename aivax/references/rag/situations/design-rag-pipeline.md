# Design RAG Pipeline

Use when the agent must design a RAG pipeline from scratch: choose the collection structure, the chunking strategy, the query strategy, the reranker, the top K, the min score, and the gateway wiring.

## Objective

Produce a pipeline that returns the right evidence for the user's queries, with the right balance of recall, precision, latency, and cost.

## Preconditions

- The user goal is clear: what questions should the pipeline answer, and what is the source of truth for the answers.
- The source documents are accessible: files, URLs, internal databases, or another storage.
- The plan and balance are known.

## Decision Tree

1. What is the source of the documents? Files, URLs, internal data, or a mix. The ingestion path differs for each.
2. How should the documents be chunked? Short chunks (a few hundred tokens) for precise retrieval; long chunks (a few thousand tokens) for context-heavy questions. Use `references/text-tools/situations/segment-text.md` to choose.
3. What metadata should the documents have? Tags, references, owners, and dates help the model reason about the source. Add metadata at ingestion time.
4. How many collections? One per domain. Splitting by domain reduces cross-contamination and improves precision.
5. What query strategy? `Plain` for direct lookups; `FullRewrite` for conversational queries; `UserRewrite` when the user often refers to earlier turns; `QueryFunction` for custom transformation.
6. What reranker? Default is `lexical` (no cost, no quota). Reflex is the default AIVAX reranker for low-latency semantic reranking. See `references/rerankers/`.
7. What top K? 5 to 10 is a good default. Larger K improves recall but inflates the prompt and the cost.
8. What min score? 0.2 is a good default. Higher is more precise; lower is more permissive.
9. How are documents updated? Single-document upsert, JSONL import, or a webhook that re-indexes on change. The update path affects freshness.
10. How is grounding evaluated? With a held-out set of questions and expected evidence. See `situations/evaluate-groundedness.md`.

## Construction

```text
1. Create a collection
   POST /api/v1/collections
   { "name": "<domain>", "context": "<what this collection is authoritative for>", "tags": "<comma-separated>" }

2. Index the documents
   PUT /api/v1/collections/<id>/documents
   { "name": "<doc-name>", "contents": "<text>", "reference": "<url>", "tags": ["..."], "metadata": { "owner": "..." } }
   or
   POST /api/v1/collections/<id>/documents (JSONL import)

3. Wait for indexing to complete (browse documents with ?state=queued to confirm)

4. Wire the collection to a gateway
   PATCH /api/v1/ai-gateways/<gateway-id>
   { "parameters": { "knowledgeCollections": ["<collection-id>"], "knowledgeBaseMaximumResults": 8, "knowledgeBaseMinimumScore": 0.2, "rerankerName": "lexical", "queryStrategy": "FullRewrite" } }

5. Validate through a test conversation
```

## Validation

- The retrieval returns the expected evidence for a sample of representative queries.
- The generated answer cites the evidence.
- The grounding score is acceptable.
- The latency is within the user's budget.
- The cost is within the user's cap.

## Failure Modes

- The retrieval returns nothing: the documents are not indexed, the collection is empty, or the query is too narrow. Inspect the collection state and the query.
- The retrieval returns irrelevant documents: the chunking is wrong, the query strategy is wrong, or the collection context is missing. Adjust.
- The answer ignores the evidence: the prompt is too strong, the model is not strong on grounding, or the evidence is presented out of order. Adjust the system instruction and the evidence ordering.
- The pipeline is too slow: lower top K, raise min score, switch to a faster reranker, or switch to a faster model.

## Escalation

- The retrieval is consistently wrong: load `situations/debug-bad-answer.md`.
- The expected document is missing: load `situations/debug-missing-evidence.md`.
- The grounding score is low: load `situations/evaluate-groundedness.md`.
- The pipeline is too expensive: load `references/cost-monitoring/situations/optimize-spend.md`.

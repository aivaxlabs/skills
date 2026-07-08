# RAG Analysis

Use this sub-skill to inspect AIVAX retrieval behavior from the account surface, not from guesses. Start with collection state and real query evidence, then decide whether the fix belongs in documents, collection context, gateway RAG settings, or user-facing instructions.

This is a platform-usage workflow. Use collections, documents, RAG queries, quality reports, transactions, and gateway settings exposed through AIVAX. Do not inspect or modify AIVAX retrieval source code.

## Situation Playbooks

- `situations/diagnose-bad-answer.md`: wrong, incomplete, hallucinated, outdated, or uncited RAG answer.
- `situations/inspect-retrieval.md`: inspect which documents were retrieved and why.
- `situations/debug-missing-documents.md`: expected source is absent, queued, stale, or not connected.
- `situations/evaluate-quality.md`: evaluate collection readiness and retrieval quality.

Use the listed API paths through `aivax_invoke_function`. If request or response fields are uncertain, call `aivax_search_context` before changing collection, document, vector, or gateway settings.

```text
aivax_invoke_function({
  "method": "POST",
  "path": "/api/v1/query",
  "body": {
    "terms": ["refund policy"],
    "collections": ["<collection-id>"],
    "top": 10,
    "minScore": 0.2,
    "reranker": "lexical",
    "includeReferences": true
  }
})
```

## Discovery Workflow

1. List collections:

```text
GET /api/v1/collections
```

2. For each relevant collection, inspect details:

```text
GET /api/v1/collections/<collection-id>
```

Review `State`, discovered `Tags`, `Context`, `ContextualTags`, and `Statistics`. Treat these as inspection targets; preserve the exact field names and casing returned by the API response when reporting or patching.

3. Browse suspicious documents:

```text
GET /api/v1/collections/<collection-id>/documents?filter=<text>
GET /api/v1/collections/<collection-id>/documents?state=queued
GET /api/v1/collections/<collection-id>/documents?order_by=updated_at_desc
```

4. Get full documents only when needed:

```text
GET /api/v1/collections/<collection-id>/documents/<document-id>
```

Prefer listings and previews first. Full document retrieval can be large.

## Retrieval Testing

Use semantic search for direct retrieval evidence:

```text
POST /api/v1/query
{
  "terms": ["user question or rewritten search phrase"],
  "collections": ["<collection-id>"],
  "top": 10,
  "minScore": 0.2,
  "reranker": "lexical",
  "includeReferences": true
}
```

Use answer generation only when evaluating end-to-end answerability:

```text
POST /api/v1/answer
{
  "terms": ["question"],
  "collections": ["<collection-id>"],
  "top": 10,
  "minScore": 0.2,
  "reranker": "lexical",
  "includeReferences": true
}
```

Test with at least three query types when possible: exact expected wording, user-like wording, and ambiguous wording. Compare document IDs, scores, content previews, and whether the answer used the right evidence.

## RAG Transaction Diagnostics

Use transaction telemetry to understand real production retrieval:

```text
GET /api/v1/collections/<collection-id>/transactions?view=recent
GET /api/v1/collections/<collection-id>/transactions?view=low-quality
GET /api/v1/collections/<collection-id>/transactions?view=high-quality
GET /api/v1/collections/<collection-id>/transactions/<transaction-id>
GET /api/v1/collections/<collection-id>/transactions/export/jsonl?view=low-quality
```

Diagnose:

- Low original score: query wording does not match indexed content or content is missing.
- Good original score but poor reranker score: reranker may be demoting useful evidence or documents are too noisy.
- Many irrelevant results: collection context/tags are weak, chunking is too broad, or min score is too low.
- No results: document indexing is incomplete, wrong collection is attached, or query strategy is unsuitable.
- High process time: top K, references, reranker, or collection size may be too expensive.

## Decision Guide

| Symptom | Evidence to collect | Likely fix | Validation |
| --- | --- | --- | --- |
| No results | Query response, collection state, queued documents | Attach the right collection, finish indexing, lower `minScore`, or improve query strategy | Repeat `/api/v1/query` and inspect a recent transaction |
| Irrelevant results | Top results, scores, tags, collection context | Improve collection context/tags, raise `minScore`, reduce noisy documents | Compare exact, user-like, and ambiguous test queries |
| Standalone query works but gateway fails | Gateway detail and conversation resources | Fix `knowledgeCollections`, top K, min score, references, or `queryStrategy` | Run a gateway test conversation and inspect the conversation record |
| Slow or costly RAG | Transaction process time, top K, references, usage | Lower top K, disable references, tune reranker/query strategy | Re-check transactions and cost monitoring |

## Quality Reports

Use built-in collection performance reports for structured quality review:

```text
POST /api/v1/collections/<collection-id>/performances
GET  /api/v1/collections/<collection-id>/performances
GET  /api/v1/collections/<collection-id>/performances/<performance-id>
POST /api/v1/collections/<collection-id>/performances/<performance-id>/cancel
```

These reports can spend account balance. Queue them intentionally, then review `OverallScore`, `OverallReadiness`, findings, and remediation plan before editing documents.

## Remediation Actions

Use the smallest correction that addresses the evidence.

- Edit collection name/context/tags:

```text
PUT /api/v1/collections/<collection-id>
{
  "collectionName": "Support Knowledge Base",
  "context": "What this collection is authoritative for.",
  "tags": "support,policy,billing"
}
```

- Upsert one document:

```text
PUT /api/v1/collections/<collection-id>/documents
{
  "name": "refund-policy",
  "contents": "Authoritative text...",
  "reference": "https://example.com/refund-policy",
  "tags": ["policy", "billing"],
  "metadata": { "owner": "support" }
}
```

- Import JSONL for many documents with multipart form data and `documents` as the file field. Use `insert-mode=sync` only when the user explicitly wants documents missing from the upload to be deleted.

- Rebuild vectors:

```text
DELETE /api/v1/collections/<collection-id>/vectors-only
```

This queues vector rebuilds without deleting documents.

Avoid reset or delete unless the user clearly asked:

```text
DELETE /api/v1/collections/<collection-id>/reset-only
DELETE /api/v1/collections/<collection-id>
```

## Gateway RAG Tuning

When retrieval fails inside a gateway but standalone `/api/v1/query` works, inspect:

- `knowledgeCollections` includes the intended collection IDs.
- `knowledgeBaseMaximumResults` is high enough for the use case.
- `knowledgeBaseMinimumScore` is not filtering useful evidence.
- `knowledgeUseReferences` matches the document structure.
- `knowledgeUseMetaDescriptions` is enabled when metadata helps ranking.
- `queryStrategy` is appropriate: `Plain`, `Concatenate`, `FullRewrite`, `UserRewrite`, or `QueryFunction`.
- `queryStrategyParameters` use enough recent context for multi-turn questions.

## Report Format

Return collections inspected, test queries, observed top results, transaction evidence, root cause category, exact changes applied or recommended, and validation performed.

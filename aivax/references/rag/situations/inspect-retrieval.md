# Inspect Retrieval

Use when the user asks what documents AIVAX retrieved or why a query matched certain content.

Goal: show retrieval evidence without guessing.

## Steps

1. Identify collection ID, original terms, and the exact `filter`, if any. Confirm whether the caller used semantic search or filter-only retrieval.
2. View collection details:

```text
GET /api/v1/collections/<collection-id>
```

3. Replay the terms and filter. For semantic search, use a low enough `minScore` to inspect candidates. For filter-only requests, results are newest first with `score: 1`; changing the reranker or `minScore` does not improve that lookup.
4. Compare exact wording, user-like wording, and ambiguous wording while keeping the filter fixed. Use a separate filter-only lookup to isolate eligibility from relevance.
5. Inspect document IDs, scores, content, metadata, and `timestamps.createdAt`/`timestamps.updatedAt`. Check `referencedDocuments` separately if included: the filter is not reapplied to them.
6. Inspect recent RAG transactions if this came from a real gateway conversation.

## Report

Include:

- Collection inspected.
- Query terms, filter, and retrieval mode used.
- Top document IDs/references.
- Score pattern.
- Whether expected evidence appeared.
- Whether issue is retrieval, ranking, or generation.

## Validation

If a change is made, rerun the same retrieval query and compare top results.


# Inspect Retrieval

Use when the user asks what documents AIVAX retrieved or why a query matched certain content.

Goal: show retrieval evidence without guessing.

## Steps

1. Identify collection ID and query terms.
2. View collection details:

```text
GET /api/v1/collections/<collection-id>
```

3. Run direct query with a low enough `minScore` to inspect candidates.
4. Compare exact wording, user-like wording, and ambiguous wording when possible.
5. Inspect document IDs, scores, previews, references, metadata, and tags.
6. Inspect recent RAG transactions if this came from a real gateway conversation.

## Report

Include:

- Collection inspected.
- Query terms used.
- Top document IDs/references.
- Score pattern.
- Whether expected evidence appeared.
- Whether issue is retrieval, ranking, or generation.

## Validation

If a change is made, rerun the same retrieval query and compare top results.


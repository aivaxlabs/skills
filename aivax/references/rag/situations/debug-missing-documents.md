# Debug Missing Documents

Use when expected knowledge is absent from RAG answers or collection search.

Goal: determine whether the document is absent, queued, failed, stale, poorly tagged, or not connected to the gateway.

## Steps

1. List collections and confirm the intended collection.
2. Browse documents with filters:

```text
GET /api/v1/collections/<collection-id>/documents?filter=<expected-term>
GET /api/v1/collections/<collection-id>/documents?state=queued
GET /api/v1/collections/<collection-id>/documents?order_by=updated_at_desc
```

3. View the document only when the listing suggests a match.
4. Check collection state, tags, context, statistics, and indexing status.
5. Check whether the gateway includes the collection ID.
6. If documents exist but retrieval fails, run `inspect-retrieval.md`.
7. If vectors are stale, consider rebuilding vectors only after confirming documents are present.

## Avoid

- Resetting or deleting a collection before proving the issue.
- Importing duplicate documents without checking existing names/references.
- Using `insert-mode=sync` unless the user explicitly wants missing upload rows deleted.

## Validation

- Confirm document listing shows the expected source.
- Run `/api/v1/query`.
- Confirm gateway retrieval if the issue was gateway-only.


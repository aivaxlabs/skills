# Debug Missing Evidence

Use when the expected document is not in the retrieval results, or when the user reports that a known fact is absent from the answers.

## Objective

Determine whether the document is missing from the collection, not yet indexed, indexed but with a poor embedding, not attached to the gateway, or attached but the query strategy does not surface it.

## Preconditions

- The collection and the gateway are known.
- A representative query that should retrieve the missing document is available.

## Decision Tree

1. Does the document exist in the collection? Browse the collection with a filter for the expected term. If the document is not in the listing, it was never indexed or it was deleted.
2. Is the document still queued or failed? Browse with `?state=queued` or `?state=error`. If it is queued, wait. If it failed, the document needs to be re-ingested with a fix.
3. Is the document's metadata rich enough? A document with no tags and no context is hard to retrieve. Add metadata at ingestion time.
4. Is the collection attached to the gateway? View the gateway and check `parameters.knowledgeCollections`. If the collection is not in the list, attach it.
5. Is the min score too high? Lower it. A high min score rejects documents that would otherwise be returned.
6. Is the top K too low? Increase it. A low top K drops relevant documents.
7. Is the query strategy wrong? `Plain` does not transform the query. For conversational or ambiguous queries, switch to `FullRewrite` or `UserRewrite`.
8. Is the reranker demoting the document? Run `/api/v1/query` with `reranker: "none"` and see if the document appears. If it does, the reranker is the problem.
9. Are the vectors stale? Rebuild vectors with `DELETE /api/v1/collections/<id>/vectors-only`. The collection documents are preserved.

## Actions

1. Browse the collection to confirm the document is indexed and the metadata is correct.
2. Run a direct semantic search (`POST /api/v1/query`) with the same query and a low min score to surface candidates.
3. Compare the candidates with the expected document. If the expected document is not in the candidates, the issue is ingestion or chunking.
4. If the document is in the candidates but the gateway does not return it, check the gateway RAG settings and the query strategy.
5. If the document is in the gateway results but the answer ignores it, the issue is generation. Load `situations/debug-bad-answer.md`.

## Validation

- A direct semantic search returns the expected document.
- The gateway's RAG settings include the collection, the right top K, the right min score, and the right query strategy.
- A test conversation through the gateway returns an answer that cites the expected document.

## Limits

- Do not reset or delete the collection to fix a missing document. Inspect first.
- Do not import duplicate documents. Check existing names and references before importing.
- Do not use `insert-mode=sync` unless the user explicitly wants missing upload rows deleted.

## Escalation

- The document is in the collection but the embedding is wrong: rebuild vectors or re-ingest the document with a fix.
- The collection context is missing or wrong: edit the collection metadata with `PUT /api/v1/collections/<id>`.
- The query strategy is not strong enough: try a different strategy or load `references/rerankers/`.
- The grounding score is low after the fix: load `situations/evaluate-groundedness.md`.

# Debug Bad Answer

Use when a RAG answer is wrong, incomplete, hallucinated, outdated, or uncited. The answer can be bad for many reasons; the right fix depends on which.

## Objective

Identify the failure layer: retrieval, ranking, source data, gateway RAG settings, context construction, or generation. Apply the smallest fix that addresses the evidence.

## Preconditions

- The original user query, the gateway, the collection, and the time of the failing answer are known.
- A test query is available that should produce a correct answer.

## Decision Tree

1. Run a direct semantic search (`POST /api/v1/query`) with the original query and a low min score. Are the expected documents in the candidates?
   - No: the issue is retrieval. Load `situations/debug-missing-evidence.md`.
   - Yes: the retrieval works. The issue is downstream.
2. With the expected documents in the candidates, is the reranker demoting them? Run `/api/v1/query` with `reranker: "none"` and see if the documents rise. If they do, the reranker is the problem. Load `references/rerankers/`.
3. Is the top K too low? Increase it. A low top K drops relevant documents before the model sees them.
4. Is the min score too high? Lower it. A high min score rejects documents that would otherwise be returned.
5. Is the query strategy rewriting the query poorly? Try `Plain` to see if the rewritten query is the problem. Switch to a different strategy.
6. Are the documents presented to the model in the right order? The model attends more to the top of the context. Order by relevance.
7. Is the system instruction telling the model to use the evidence? Tighten the system instruction to require citations.
8. Is the model strong on grounding? Switch to a model that is known to follow evidence.
9. Is the answer cached from a previous configuration? Wait for the cache to expire or restart the gateway.

## Actions

1. Run a direct semantic search with the original query and a low min score. Capture the candidates.
2. Compare the candidates with the expected documents. If the expected documents are not in the candidates, the issue is ingestion or chunking.
3. If the expected documents are in the candidates but not in the gateway's top K, adjust the top K and the min score.
4. If the gateway's top K includes the expected documents but the answer ignores them, the issue is generation. Tighten the system instruction and switch to a model that is stronger on grounding.
5. Run a test conversation through the gateway to validate.

## Validation

- The answer cites the expected documents.
- The answer is faithful to the source.
- The grounding score is acceptable. See `situations/evaluate-groundedness.md`.
- The latency is within the user's budget.
- The cost is within the user's cap.

## Failure Modes

- The answer is correct but the user expected more detail: increase top K or lower min score.
- The answer cites a wrong document: the chunking is wrong or the collection context is missing. Inspect the cited document.
- The answer is correct on a small sample but wrong on a larger sample: the query strategy is not generalizing. Switch to a stronger strategy or a stronger model.

## Escalation

- The grounding score is low: load `situations/evaluate-groundedness.md`.
- The reranker is the problem: load `references/rerankers/`.
- The model is the problem: load `references/text-inference/situations/choose-model.md`.
- The pipeline is too expensive: load `references/cost-monitoring/situations/optimize-spend.md`.

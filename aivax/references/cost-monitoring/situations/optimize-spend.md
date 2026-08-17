# Optimize Spend

Use when the user wants to reduce AIVAX spend safely, without harming answer quality or user experience, and without changing customer-facing behavior.

## Objective

Produce a prioritized list of safe optimizations, ordered by savings and behavior risk, that the user can approve one at a time.

## Preconditions

- The user is willing to consider changes to the production configuration.
- The agent has a recent usage view (24h, 7d, 30d) to inform the recommendations.
- The agent has the resource IDs and configurations that drive the spend.

## Decision Tree

1. Identify the dominant cost driver. Load `situations/investigate-cost-spike.md` if the cause is not yet clear.
2. For each driver, list the safe levers. The lever is safe when it does not change user-visible behavior at the agreed quality bar.
3. Order the levers by savings (high to low) and by behavior risk (low to high).
4. The user approves one or more levers. The agent applies them one at a time and validates.
5. Re-check the next usage window to confirm the cost delta.

## Levers By Resource

Model and gateway:

- Replace overpowered models with cheaper compatible models.
- Use model routing when request complexity varies.
- Lower `maxCompletionTokens` when outputs are longer than needed.
- Use `contextMaximumSize` and an appropriate `contextOverflowAction`.
- Keep `reasoningEffort` proportional to task complexity.

RAG:

- Lower `knowledgeBaseMaximumResults` when extra documents do not improve answers.
- Raise `knowledgeBaseMinimumScore` when irrelevant documents inflate context.
- Disable `knowledgeUseReferences` unless referenced parents materially improve grounding.
- Tune `queryStrategy` instead of blindly increasing retrieval volume.
- Use collection transactions and quality reports before reindexing or rewriting large corpora.

Tools:

- Enable only necessary built-in functions, MCP tools, and protocol functions.
- Use `hideToolsWithoutSkill` with carefully designed skills for tool-heavy gateways.
- Investigate repeated tool calls in conversations before raising tool context or model limits.
- Limit `toolContextCount` when old tool results inflate prompts.

Chat clients and sessions:

- Set sensible `messagesPerHour` and `maxMessages`.
- Avoid large repeated `extraContext`; use concise session metadata.
- Disable debug modes and unnecessary upload modes after troubleshooting.
- Review integration debounce and scheduled-continuation settings if channel usage spikes.

Batch:

- Run a small sample before activating large jobs.
- Use strict result schemas and validation only when the business value justifies the extra calls.
- Export low-confidence and error items before retrying.
- Retry targeted modes rather than all failed items.

Storage:

- Use collection details and document browsing to find stale collections or oversized corpora.
- Prefer targeted document cleanup over collection reset.
- Delete collections only with explicit user approval and export if retention matters.

## Approval Gates

Get explicit approval before:

- Model swaps in production.
- Pausing or finishing active jobs.
- Lowering limits.
- Deleting storage or collections.
- Restricting tools or skills.
- Changing customer-facing chat behavior.

## Validation

- The optimization is applied in isolation (one change at a time when possible).
- The next usage window reflects the expected cost delta.
- The quality bar is still met. Spot-check a few representative conversations.
- The change is recorded in the change log.

## Limits

- Some optimizations (storage cleanup, batch finish, model swap) are destructive. Confirm with the user.
- A change that affects customer-facing behavior requires explicit user approval.
- A change that lowers the quality bar is not safe; do not recommend it.

## Escalation

- A model swap is recommended: load `references/text-inference/situations/choose-model.md`.
- A RAG tuning is recommended: load `references/rag/situations/design-rag-pipeline.md`.
- A batch retry is recommended: load `references/batch/situations/debug-failed-job.md`.
- A storage cleanup is recommended: confirm with the user and export before deletion.

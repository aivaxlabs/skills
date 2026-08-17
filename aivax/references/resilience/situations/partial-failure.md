# Partial Failure

Use when a chain of AIVAX calls fails mid-way. Some stages have completed; others have not. The agent must decide whether to roll back, skip the failed stage, or report partial state.

## Objective

Leave the account in a state that is either fully consistent with the user's goal, fully rolled back, or explicitly marked as partial — never ambiguous.

## Preconditions

- A pipeline of AIVAX calls was executing.
- One stage failed; earlier stages may have completed.

## Decision Tree

1. Is the failed stage's effect reversible? If yes, roll back the completed stages. This is the safest path.
2. Is the failed stage optional (the pipeline can still meet the user goal without it)? Skip it. Document the skip.
3. Is the failed stage's effect not reversible (e.g. an external side effect, a sent message, a published web chat talk URL)? The pipeline cannot be rolled back. Mark the state as partial and surface the partial result to the user.
4. Is the failed stage's effect idempotent (re-running it produces the same outcome)? Retry. See `situations/idempotent-retry.md`.
5. None of the above? Stop. Do not continue. Surface the partial state and ask the user.

## Actions

1. Capture the exact stage that failed, the error class, and the resource IDs that were created or modified before the failure.
2. For each completed stage, determine if the effect is reversible, optional, or permanent.
3. If rolling back, capture the pre-state for each completed stage (the pipeline should have recorded this; if not, the rollback is unsafe — stop and surface).
4. If skipping, document the skipped stage and the impact on the user-facing result.
5. If marking partial, list exactly which stages completed and which did not, and what the user should do.
6. Update the trace with the failure class, the decision, and the IDs involved.

## Validation

- The account state is one of: rolled back, partial-but-marked, or completed with a documented skip.
- The trace is updated and links to the failed stage.
- The user is informed in plain language of the final state.

## Limits

- Do not continue a failed pipeline hoping the next stage will fix the problem. The next stage usually compounds the failure.
- Do not roll back a stage that has external side effects. Surface instead.
- A pipeline that frequently needs rollback is a pipeline that needs redesign. Add pre-conditions and validation between stages.

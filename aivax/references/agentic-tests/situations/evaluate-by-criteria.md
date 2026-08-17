# Evaluate By Criteria

Use when the agent must evaluate a run against a criteria with a judge model, or when the user wants a structured score for a test.

## Objective

Produce a per-input score and an overall score for a run, with the judge's reasoning recorded for audit.

## Preconditions

- A run is complete.
- A criteria is defined (either in the test or as a separate evaluation call).
- A judge model is available. A stronger judge model produces more consistent scores; a cheaper judge model saves cost.

## Decision Tree

1. Is the criteria already in the test? Use it. The evaluation call uses the test's criteria.
2. Is the criteria a new one? Define it on the evaluation call.
3. Which judge model? A reasoning model with strong instruction following is best. A small model can be a judge for narrow criteria.
4. What is the score scale? Binary (pass / fail), 1-5, or 0-1. The scale must be consistent across runs.
5. Is the evaluation deterministic? Set `temperature: 0` on the judge (or the judge's equivalent).
6. How is the reasoning recorded? The judge produces a per-input score and a reasoning. Store the reasoning for audit.

## Construction

```text
POST /api/v1/agentic-tests/<test-id>/evaluate
{
  "runId": "<run-id>",
  "criteria": "<evaluation-criteria>",
  "judgeModel": "<judge-model>",
  "metadata": { "trace_id": "tr_..." }
}
```

The exact field shape depends on the AIVAX version. Verify with `aivax_search_context` before relying on a field.

## Validation

- The evaluation produces a per-input score and an overall score.
- The judge's reasoning is recorded for audit.
- The trace ID is preserved.
- The score is consistent across re-runs (when the judge is deterministic).

## Failure Modes

- The judge produces inconsistent scores: the criteria is vague or the judge is probabilistic. Tighten the criteria and set `temperature: 0`.
- The judge's score disagrees with a human review: the judge is not strong on the domain or the criteria is wrong. Spot-check.
- The evaluation is too expensive: the judge model is too large or the criteria is too long. Use a smaller judge or a shorter criteria.

## Escalation

- The criteria is too vague: load `situations/design-test-case.md` and tighten the criteria.
- The judge model is too small: load `references/text-inference/situations/choose-model.md` and pick a stronger judge.
- The run needs to be re-evaluated: load `situations/run-and-collect-traces.md` and re-run the test.

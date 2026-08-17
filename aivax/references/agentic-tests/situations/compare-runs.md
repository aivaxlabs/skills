# Compare Runs

Use when comparing repeated conversational runs for regression, improvement, reliability, cost, or behavioral drift.

## Preconditions

Retrieve complete run details with:

```http
GET /api/v1/agentic-tests/{test-id}/runs/{run-id}
```

Prefer runs from the same persisted test definition. A run does not store a snapshot of every definition field, so if the test was edited between runs, the API alone may not prove which gateway, goal, criteria, or thresholds each run used. Record definition snapshots externally when strict reproducibility matters.

## Comparable Dimensions

Compare only implemented data:

- operational `state` and `error`;
- behavioral `result.outcome`, `result.reason`, and `result.state`;
- final top-level `score` / `result.conversation_delta`;
- `turn_number` and `loss_streak`;
- total `cost`;
- retained user, assistant, and judge conversation;
- per-message prompt, cached prompt, and completion tokens.

There are no per-input scores, input flips, dataset sample size, built-in variance, latency metric, or statistical-significance field.

## Procedure

1. **List candidates**

   ```http
   GET /api/v1/agentic-tests/{test-id}/runs?limit=100&offset=0
   ```

   Optionally filter by one exact state.

2. **Fetch complete details** for the selected run IDs.
3. **Check comparability**: confirm the test definition was unchanged or disclose known changes. At minimum compare gateway, goal, criteria, start conversation, turn controls, thresholds, profile, and sampling.
4. **Separate operational reliability from behavior**:
   - `failed` versus terminal execution without error;
   - `success`, `loss`, or `incomplete` among `succeeded` runs.
5. **Compare trajectories**: inspect judge reasoning and conversation evidence, not only the final score.
6. **Compare consumption**: report total cost and token differences. Do not claim latency unless independently measured from timestamps, and label that calculation as derived.
7. **Classify cautiously**: one stochastic conversation is evidence, not statistical proof. Repeat a controlled definition when confidence matters.

## Compact Report

| Dimension | Baseline | Candidate |
| --- | --- | --- |
| Run ID | `{id}` | `{id}` |
| Execution state | `succeeded` | `succeeded` |
| Outcome / reason | `success / baseline_reached` | `incomplete / max_turns_reached` |
| Final trajectory | `0.94` | `0.62` |
| Turns | `4` | `10` |
| Cost | `...` | `...` |
| Key judge evidence | `...` | `...` |

Then state whether the evidence suggests a regression, improvement, operational failure, or inconclusive variation, and name any definition change that could explain it.

## Repeated Runs

To estimate consistency, queue multiple runs intentionally and compare the distribution of outcomes and scores yourself. Agree on run count and cost before doing so. Do not automatically launch three or more paid runs, and do not claim statistical significance from a small unplanned sample.

## Common Errors

- Comparing `succeeded` counts as pass rates without reading `result.outcome`.
- Claiming per-input regressions; each run is one simulated conversation.
- Assuming old runs preserve the exact historical test definition.
- Comparing a `failed` provider execution to a behavioral `loss` as if they were the same failure.
- Recommending rollback based on one stochastic run without checking conversation evidence and repeatability.

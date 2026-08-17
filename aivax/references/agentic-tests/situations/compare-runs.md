# Compare Runs

Use when the agent must compare two or more runs to detect regressions, improvements, or behavior changes.

## Objective

Identify what changed between runs and whether the change is a regression, an improvement, or a neutral variation. The comparison should be auditable and reproducible.

## Preconditions

- Two or more runs are complete.
- The runs target the same test (or compatible tests).
- The runs were executed under the same evaluation criteria (or the criteria are aligned).

## Decision Tree

1. Are the runs on the same target? If not, the comparison is across targets (model vs. model, gateway vs. gateway). Useful for swap decisions.
2. Are the runs on the same input set? If not, the comparison is across input sets, which is harder to interpret.
3. Are the runs on the same evaluation criteria? If not, align the criteria before comparing.
4. Is the comparison for a regression? Compare the latest run to a known-good baseline. A drop in score is a regression.
5. Is the comparison for an improvement? Compare the new run to the previous run. A rise in score is an improvement.
6. Is the difference statistically meaningful? A small sample can produce noise. Run the test multiple times and inspect the variance.
7. Is the change in one input or many? A change in many inputs is more meaningful than a change in one.

## Construction

```text
1. List runs
   GET /api/v1/agentic-tests/<test-id>/runs

2. For each pair of runs (baseline, candidate):
   - Per-input score diff
   - Overall score diff
   - Number of inputs that flipped (pass -> fail, fail -> pass)
   - Cost and latency diff

3. Classify the change
   - Regression: overall score dropped, flips to fail outnumber flips to pass
   - Improvement: overall score rose, flips to pass outnumber flips to pass
   - Neutral: no meaningful change

4. Identify the cause
   - The target changed (model swap, gateway edit)
   - The instruction changed
   - The input set changed
   - The evaluation criteria changed
   - The model is probabilistic and the change is variance

5. Report
   - The diff per metric
   - The classification
   - The likely cause
   - The recommended next step
```

## Validation

- The comparison is on the same test (or compatible tests).
- The criteria are aligned.
- The statistical significance is reported (variance, sample size, confidence).
- The trace ID is preserved.
- The report is reproducible.

## Failure Modes

- The runs are not comparable: the target, input set, or criteria differ. Align before comparing.
- The difference is variance: the model is probabilistic. Run the test multiple times and report the variance.
- The cause is unclear: a change in the target, instruction, input set, or criteria was not recorded. The comparison is unauditable.

## Escalation

- A regression is detected: roll back the change that caused it. Load `references/platform-rules/safe-mutations.md`.
- An improvement is detected: confirm it is not a fluke. Re-run the test and compare again.
- The variance is high: increase the sample size or set `temperature: 0` (if supported by the target).

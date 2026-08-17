# Run And Collect Traces

Use when the agent must run an agentic test, collect the per-input outputs, and inspect the traces.

## Objective

Run a test end to end, capture the outputs and traces for analysis, and surface the cost and the duration of the run.

## Preconditions

- A test is created and is in a runnable state.
- The cost cap is set with the user.
- The trace ID is generated.

## Decision Tree

1. Is the test small (fewer than 50 inputs)? Run it directly. Inspect the run while it is in progress.
2. Is the test large? Consider running it in batch through `references/batch/`. The async nature of batch matches the async nature of long tests.
3. Is the test flaky? Run it three times and inspect the variance.
4. Is the test taking too long? Cancel the run and split the test into smaller pieces.
5. Is the test producing noise? Inspect a few representative inputs and decide whether to abort.

## Construction

```text
1. Run the test
   POST /api/v1/agentic-tests/<test-id>/run
   { "metadata": { "trace_id": "tr_..." } }

2. List runs
   GET /api/v1/agentic-tests/<test-id>/runs

3. View a run
   GET /api/v1/agentic-tests/<test-id>/runs/<run-id>

4. Inspect per-input outputs, traces, errors, cost
```

## Validation

- The run completes without error.
- The cost is within the cap.
- The trace ID is preserved.
- The per-input outputs and traces are inspectable.

## Failure Modes

- The run is cancelled: the cost is still charged for the completed inputs. There is no rollback.
- The run produces no output for an input: the target is wrong, the input is invalid, or the target is rate-limited. Inspect the trace.
- The run is flaky: the target is probabilistic. Run it multiple times and inspect the variance.

## Escalation

- The run is too slow: load `references/batch/` and consider running the test in batch.
- The run is too expensive: load `references/cost-monitoring/situations/optimize-spend.md`.
- The run is producing poor outputs: load `situations/design-test-case.md` and check the instruction.

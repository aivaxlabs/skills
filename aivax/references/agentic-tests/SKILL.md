---
name: aivax-agentic-tests
description: Create, run, evaluate, and compare agentic tests on AIVAX — test cases, datasets, traces, criteria-based evaluation, run management, and failure analysis. Load when the agent must verify a model's behavior on a held-out set, compare models or prompts, or audit a gateway's outputs over time.
---

# Agentic Tests

This sub-skill owns AIVAX agentic tests. An agentic test is a reusable definition of an evaluation: a name, a target (a model, a gateway, or a chat client), an instruction set, an input set, and an evaluation criteria. A run is a single execution of the test. Evaluation is a structured judgment of the outputs against the criteria.

## Operating Files

- `situations/design-test-case.md`: design a test case that captures a specific behavior.
- `situations/run-and-collect-traces.md`: run a test, collect traces, and inspect the output.
- `situations/evaluate-by-criteria.md`: evaluate a run against criteria, with a judge model.
- `situations/compare-runs.md`: compare two or more runs to detect regressions or improvements.

## Endpoints

- `GET /api/v1/agentic-tests`: list tests.
- `GET /api/v1/agentic-tests/<test-id>`: view a test.
- `POST /api/v1/agentic-tests`: create a test.
- `PATCH /api/v1/agentic-tests/<test-id>`: update a test.
- `DELETE /api/v1/agentic-tests/<test-id>`: delete a test.
- `POST /api/v1/agentic-tests/<test-id>/evaluate`: evaluate a run.
- `POST /api/v1/agentic-tests/<test-id>/run`: run the test.
- `GET /api/v1/agentic-tests/<test-id>/runs`: list runs.
- `GET /api/v1/agentic-tests/<test-id>/runs/<run-id>`: view a run.
- `POST /api/v1/agentic-tests/<test-id>/runs/<run-id>/cancel`: cancel a run.

## When To Use Agentic Tests

Use this sub-skill when the agent must:

- Verify that a model or gateway produces a specific output for a specific input.
- Compare two models, two prompts, or two gateways on the same input set.
- Detect regressions after a gateway, prompt, or model change.
- Audit a gateway's outputs over time against a known-good baseline.
- Evaluate the quality of a generation against a criteria (e.g. "answer is grounded", "answer is concise").

Do not use this sub-skill for:

- A single one-off check. Use `references/text-inference/` or `references/observability/`.
- A performance report on a RAG collection. Use `references/rag/situations/evaluate-groundedness.md`.

## Test Components

A test has:

- A target: a model name, a gateway ID, or a chat client.
- An instruction: the system prompt or the gateway's instruction set.
- An input set: a list of inputs to run.
- An evaluation criteria: a description of what makes a good output.
- Optional: tools, expected outputs, evaluation thresholds.

## Run Lifecycle

1. **Create the test** with a target, instruction, input set, and criteria.
2. **Run the test**. The run produces traces and outputs.
3. **Inspect the run** to see the per-input output, the trace, and any errors.
4. **Evaluate the run** against the criteria. The evaluation produces a per-input score and an overall score.
5. **Compare runs** across model changes, prompt changes, or time.
6. **Cancel a run** if it is taking too long or producing noise.

## Cost Awareness

Tests consume balance, request quota, and possibly token-rate quota. A test with 1,000 inputs can cost more than a single inference call. Set a cost cap with the user before running large tests.

## Validation

- The test definition is valid.
- The run completes without error (or is cancelled cleanly).
- The evaluation produces a score for each input.
- The trace ID is preserved.

## Limits

- A test can have many inputs but each run consumes balance per input. Set a cap.
- The evaluation criteria is qualitative. Use LLM-as-judge with care; a small sample of human review is recommended for ground truth.
- The run is async. Plan for the time it takes to complete.

## Escalation

- The test fails consistently: load `situations/design-test-case.md` and check the criteria.
- The evaluation score is low: load `situations/evaluate-by-criteria.md` and consider a stronger judge model.
- A regression is detected: load `situations/compare-runs.md` and identify the change that caused it.
- The test is too large: load `references/batch/` and consider running the test in batch.

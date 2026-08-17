# Design Test Case

Use when the agent must design a new agentic test that captures a specific behavior.

## Objective

Produce a test that is reproducible, isolated, and discriminating — a test that fails when the behavior is broken and passes when it is correct.

## Preconditions

- The behavior to test is clear (e.g. "the model should refuse to answer", "the model should cite the source", "the model should use the tool").
- A target is known: a model, a gateway, or a chat client.
- A representative input set can be built or is available.

## Decision Tree

1. Is the behavior deterministic or probabilistic? Deterministic behaviors (e.g. "the model should always refuse this prompt") need fewer inputs; probabilistic behaviors (e.g. "the model should be helpful") need more.
2. What is the unit of evaluation? Per-input score, per-run score, or both.
3. What is the criteria? A short description of what makes a good output. The criteria is the input to the judge model.
4. What is the input set? Real user queries, synthetic queries, or a mix. Real is best; synthetic is faster to build.
5. What is the target? A model name, a gateway ID, or a chat client. A gateway is the right target when the test should cover the full production configuration.
6. What is the expected output? A literal string, a JSON schema, or a description. The expected output is optional but helps the judge.

## Construction

```text
POST /api/v1/agentic-tests
{
  "name": "<test-name>",
  "target": "<model-name-or-gateway-id>",
  "instruction": "<system-prompt-or-instruction-set>",
  "inputs": [
    { "input": "<input-1>", "expected": "<expected-output-1>" },
    { "input": "<input-2>", "expected": "<expected-output-2>" }
  ],
  "criteria": "<evaluation-criteria>",
  "metadata": { "trace_id": "tr_..." }
}
```

The exact field shape depends on the AIVAX version. Verify with `aivax_search_context` before relying on a field.

## Validation

- The test definition is valid.
- A small smoke run produces a sensible output for a representative input.
- The criteria is unambiguous.
- The trace ID is preserved.

## Failure Modes

- The test definition is rejected: the target is invalid, the instruction is too long, or the inputs are not in the right shape. Inspect the error.
- The smoke run produces a wrong output: the instruction is wrong or the target is the wrong model. Adjust.
- The criteria is too vague: the judge model will produce inconsistent scores. Tighten the criteria.

## Escalation

- The behavior is not testable as a single test: split into multiple tests, each with a narrower behavior.
- The input set is too small: load `situations/evaluate-by-criteria.md` and consider a held-out set.
- The target is a chat client: load `references/web-chat/` for the chat client lifecycle.

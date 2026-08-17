# Design Pipeline

Use this situation when the task clearly spans more than one AIVAX sub-skill and the agent needs to design the pipeline before executing it.

## Objective

Produce a pipeline that meets the user goal, is auditable, fails safely, and can be retried or substituted without re-design.

## Preconditions

- The user goal is stated in user-facing terms, not in API calls.
- The dominant capability(ies) are identified. If not, return to the SKILL.md router.
- The agent has access (directly or via MCP) to the relevant AIVAX account.

## Signals

- The user describes a multi-step outcome ("first classify, then retrieve, then answer").
- Two or more sub-skills match the intent in the router.
- A previous attempt failed in a way that a different sub-skill could prevent.
- The user wants the pipeline reusable for similar tasks.

## Decision Tree

1. Is the goal achievable with a single sub-skill? If yes, do not compose. Load the sub-skill instead.
2. Is the goal achievable by composing sub-skills that each have a clear contract? If yes, continue.
3. Are the contracts compatible (input shape, output shape, error class)? If any is not, redefine the boundary; do not patch inside a sub-skill.
4. Does the pipeline have a measurement plan (eval test, conversation review, latency budget, cost cap)? If no, add it before execution.
5. Does any stage require destructive or customer-facing mutations? If yes, route to `references/platform-rules/safe-mutations.md` and confirm approval before execution.

## Actions

1. Write the pipeline sketch from the contract template in `SKILL.md`.
2. For each stage, resolve the resources (gateway ID, collection ID, reranker name, model name) the stage needs. Resolve by name, then read the resource to capture the ID.
3. For each stage, define the substitution boundary and the dominant alternative. If there is no realistic alternative, say so.
4. For each stage, define the failure class and the isolation rule. If the failure of a stage can be skipped without breaking the contract, document the skip semantics.
5. Define the cost estimate per stage and the total. The estimate does not need to be precise; it needs to be honest.
6. Define the validation step. Prefer an `agentic-tests` evaluation if the output is structural (intent, score, fields); prefer a conversation review if the output is conversational.
7. Define the approval gates per stage. Most pipelines have one or two; if there are more than three, the design is too aggressive.
8. Execute. Observe. Re-estimate cost. Compare to the evaluation. Report.

## Validation

- Each stage's output matches its declared contract.
- The chain's trace ID is recorded and propagates correctly.
- The eval test (if any) passes with a measurable margin.
- The cost estimate was within an order of magnitude of the real cost. If not, adjust the model.
- The user can answer: "what changed, what was verified, what risk remains?"

## Escalation

- A sub-skill fails in a way its own situation playbook cannot resolve: load that sub-skill's situation playbook.
- A failure class is not in the table above: load `references/resilience/` and define a new isolation rule.
- A new capability is needed that the router does not cover: stop and ask the user to confirm the new sub-skill exists.

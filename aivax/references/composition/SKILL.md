---
name: aivax-composition
description: Combine multiple AIVAX sub-skills (inference, RAG, rerankers, speech, image, agentic tests, account, observability) into a single workflow with explicit contracts, observability, and substitutable components. Load when the task spans more than one capability and the agent needs a clear pipeline.
---

# Composition

Most real work on AIVAX is not a single call. A support assistant might classify an incoming message, retrieve from a collection, rerank, generate a response, evaluate it, and log a trace. Each of these steps is owned by a different sub-skill. Composition is the contract that holds the pipeline together.

## Operating Files

- `situations/design-pipeline.md`: how to design a multi-step AIVAX workflow from scratch.
- `situations/cross-skill-observability.md`: how to keep trace IDs, request IDs, and resource IDs correlated across steps.

## When Composition Applies

Compose sub-skills when any of these are true:

- The user-facing outcome requires more than one capability (e.g. classify, then retrieve, then generate, then evaluate).
- A failure in one step must be isolable without losing the others.
- The same workflow may be retried with a different model, reranker, or collection.
- The agent needs to report cost, latency, or quality across the whole pipeline, not per call.

Do not compose when a single sub-skill solves the task. Forcing a pipeline adds cost, latency, and failure surface.

## Pipeline Contract

Every composed workflow declares the same fields. They are not optional.

- **Goal**: the user-facing outcome the pipeline exists to produce.
- **Inputs**: the data the pipeline starts with. Inputs are explicit; no implicit context unless it is in the inputs.
- **Stages**: the ordered list of sub-skills and their order. Each stage has a name, the sub-skill that owns it, the inputs it consumes, and the outputs it produces.
- **Contracts between stages**: how a stage's outputs map to the next stage's inputs. The contract is the substitution boundary: change the implementation of one stage without touching the others as long as the contract is preserved.
- **Substitutable components**: which stages have a known alternative (different model, different reranker, different gateway) and what changes when the alternative is used.
- **Observability**: which IDs are propagated (request ID, conversation ID, transaction ID, item ID) and where they are recorded.
- **Cost estimate**: per-stage cost, total cost, and the dominant driver. Re-evaluate after the pipeline runs against real data.
- **Failure handling**: per-stage failure class, isolation rule, and retry policy. Use `references/resilience/` for the retry mechanics.
- **Validation**: how the final output is checked before delivery (grounding check, eval test, conversation review, export diff).
- **Approval gates**: which stages require explicit user approval before they run (e.g. deletion, model swap, salt roll).

## Pipeline Sketch

The shape is generic; the contents depend on the goal.

```text
goal: <user outcome>
stages:
  - name: classify
    skill: references/text-tools/
    input:  <initial input>
    output: { intent, confidence }
  - name: retrieve
    skill: references/rag/
    input:  { intent.query, intent.filters }
    output: { documents: [ { id, score, text } ] }
  - name: rerank
    skill: references/rerankers/
    input:  { query: intent.query, documents: documents }
    output: { documents: [ { id, score, text } ] }
  - name: generate
    skill: references/text-inference/
    input:  { messages, context: rerank.documents }
    output: { text, citations, usage }
  - name: evaluate
    skill: references/agentic-tests/
    input:  { input, output, criteria }
    output: { score, pass }
  - name: observe
    skill: references/observability/
    input:  { all stage outputs }
    output: { trace, cost, latency }
contracts:
  - between: [ classify, retrieve ]
    map: intent -> retrieve.input.query
  - between: [ retrieve, rerank ]
    map: documents -> rerank.input.documents
substitutable:
  - stage: generate
    alternatives: [ @openai/gpt-5-mini, @google/gemini-2.5-flash, <gateway-id> ]
    cost_delta: typical_2x_to_10x
failure:
  - stage: retrieve
    class: empty
    action: skip_rerank, set confidence=low
  - stage: generate
    class: timeout
    action: retry_with_fallback_model (see resilience)
```

## Substitution Discipline

A stage is substitutable when its contract is. To substitute:

1. Confirm the new component's contract matches the stage's contract exactly (input shape, output shape, error class).
2. Re-run the validation step.
3. Update the cost estimate.
4. Re-evaluate against the same `agentic-tests` evaluation if one exists.

Do not substitute silently. Record the change so the pipeline's behavior is auditable.

## Cross-Skill Observability

Every stage must propagate a trace ID and a parent request ID. The trace ID is generated at pipeline start. Each stage adds its own resource IDs (conversation, transaction, item, run, batch, request). The final report links the whole chain.

See `situations/cross-skill-observability.md` for the propagation rules.

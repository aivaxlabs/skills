# Choose Model

Use when the agent must select (or recommend) a model for a text-inference task.

## Objective

Pick the cheapest model that meets the task's capability, latency, and quality floors — or recommend a swap with a clear rationale.

## Preconditions

- The task's required capabilities are known: modality, context, tool use, structured output, reasoning, language.
- The plan and balance are known. (See `references/account/`.)
- The agent has access to `aivax_list_models` (MCP) or `/v1/models` (fallback).

## Decision Tree

1. What is the task's modality floor? Text only, or does it require image, audio, video, or file inputs?
   - If the task includes media, the model must support that modality or `multimodal_preprocess` must be set. See `references/multimodal/`.
2. Does the task need function calling or tool use?
   - If yes, the model must declare `tools: true` in the catalog.
3. Does the task need structured output (JSON)?
   - If yes, the model must declare `structured_output: true` (or accept `response_format` with `json_schema`).
4. What is the minimum context window? Pick the smallest model whose window exceeds the prompt + expected completion + RAG injection + tool results. Plan caps may apply.
5. What is the latency budget?
   - Real-time voice: fast class.
   - Interactive chat: fast or medium class.
   - Background batch: any class.
6. What is the quality bar?
   - Hard reasoning or code: high intelligence class.
   - General chat or extraction: medium class.
   - Cheap classification or routing: low class.
7. Among the candidates that pass all floors, pick the cheapest. If two models tie on cost, pick the one with the higher intelligence class for the same speed.
8. If the user has a cost cap, narrow by pricing. If the user has a quality cap, narrow by intelligence class.

## Recommendation Output

Always include:

- The chosen model name and provider.
- A one-line rationale that names the binding constraint (cheapest with reasoning, fastest with vision, etc.).
- Expected cost per call (input + output token estimate).
- Expected latency class.
- The dominant alternative and when to switch to it.

## Validation

- The chosen model is available on the user's plan (`plan_availability`).
- The model supports every required capability (modality, tools, structured output, reasoning, context).
- A small smoke test confirms the model produces a valid response for a representative input.
- The cost estimate is within the user's cap.

## Escalation

- The model catalog has no candidate that meets the floors: surface the binding constraint, ask the user to relax it (quality, modality, latency, cost) or accept that the task cannot run.
- The recommended model is deprecated or preview: load `references/account/situations/secret-hygiene.md` and `references/cost-monitoring/situations/optimize-spend.md` for the rotation plan.
- A swap is needed in production: load `references/platform-rules/safe-mutations.md` and confirm approval before changing the gateway's model.

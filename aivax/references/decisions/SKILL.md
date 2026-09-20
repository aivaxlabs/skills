---
name: aivax-decisions
description: Evaluate named noul, choice, and score questions against a shared state with a semantic decision model. Load when the agent must classify, route, or score a text in one billed call instead of running a chat completion.
---

# Decisions

This sub-skill owns semantic decisions: one `POST` evaluates multiple named questions against a single shared `state` string and returns structured answers plus billed usage. It is a classifier/router/scorer, not a chat model — there is no message history, no tools, and no free-text reply.

## When To Use

Use this sub-skill when the agent must evaluate one text against one or more fixed criteria in a single call: boolean triage (`noul`), picking one of several named categories (`choice`), or grading intensity on an ordered scale (`score`).

Do not use this sub-skill for:

- Free-text generation, summarization, or multi-turn conversation. Use `references/text-inference/` instead.
- Document classification with grounding evaluation. Use `references/text-tools/` instead.
- Searching or reranking a knowledge base. Use `references/rag/` or `references/rerankers/` instead.

## Endpoints

- **Evaluate Decisions** (`POST /api/v1/generations/decisions`): the only endpoint. RAW envelope (no wrapping). Requires positive balance (`402` otherwise).

Request shape:

```json
{
    "model": "@typesafe/jev-1.13",
    "state": "Help! My payouts have been failing for 3 days.",
    "questions": {
        "is_urgent": {
            "type": "noul",
            "instructions": "Does this message convey urgency?",
            "criteria": { "true": "Explicitly time-sensitive", "false": "No urgency expressed" }
        },
        "department": {
            "type": "choice",
            "instructions": "Which team should handle this?",
            "criteria": { "billing": "Money, payouts, invoices", "support": "General help requests" }
        },
        "frustration": {
            "type": "score",
            "instructions": "How frustrated is the customer?",
            "criteria": ["Calm", "Frustrated", "Very angry"]
        }
    }
}
```

Rules: `model` is a string (`@` prefix optional). `state` is a non-empty string evaluated by every question. `questions` is a non-empty object keyed by question name (names cannot be empty). Each question needs non-empty `instructions` plus `criteria`: an object with `true`/`false` strings for `noul`, a non-empty object mapping choice names to descriptions for `choice`, or a non-empty ordered string array for `score` (index 0 is the top of the scale). Any other `type` returns 400.

Response shape (`id`, `model`, `provider`, `answers` keyed by question name, `usage`):

```json
{
    "id": "d_abc123def456",
    "model": "@typesafe/jev-1.13",
    "provider": "openrouter",
    "answers": {
        "is_urgent": { "type": "noul", "noul": 0.97, "confidence": 0.94 },
        "department": { "type": "choice", "choice": "billing", "probabilities": { "billing": 0.91, "support": 0.09 } },
        "frustration": { "type": "score", "score": 0.82, "legend": { "0": "Calm", "1": "Frustrated", "2": "Very angry" } }
    },
    "usage": { "input_tokens": 120, "output_tokens": 15, "cost": 0.00042 }
}
```

Answer fields by type: `noul` carries `noul` (numeric toward true/false); `choice` carries `choice` (selected identifier); `score` carries `score` plus `legend` (level identifiers to descriptions, when supplied). `probabilities` (keyed by choice or level identifier) and `confidence` appear only when the provider supplies them — never assume they are present. On `score`, interpret the numeric value against the returned `legend`: level order in the response is provider-defined, so do not infer scale direction from the request order alone. `usage` always carries `input_tokens`, `output_tokens`, and the billed `cost`. The response `model` is the canonical model name (`model.Names [ 0 ]`), which can differ from the `model` string sent in the request.

## Cost Awareness

- One call bills once for all questions combined: the response `usage.cost` is the billed amount, recorded under `serviceusage.semantic_decisions.in` (input tokens) and `serviceusage.semantic_decisions.out` (output tokens). Batching related questions into one request is cheaper than one call per question.
- Prefer decisions over a chat completion when the output is a label, a pick, or a number rather than prose: a decision call consumes far fewer output tokens.
- Record the decision `id`, model, and `usage` (tokens + cost) alongside the answers so cost spikes stay attributable. For spike analysis, load `references/cost-monitoring/`.

## Validation

- Confirm every requested question name appears in `answers` and each answer `type` matches the requested `type` before acting on the result.
- Treat a missing `probabilities`/`confidence` as "not supplied", not as zero. Do not threshold on a field that is absent.
- On `score`, interpret the value against the returned `legend`, not against the request order from memory.
- The trace ID is preserved.

## Limits

- Unknown `type` values, missing/empty `model`, `state`, `instructions`, or `criteria`, and empty question names or choice names all return 400. Fix the payload; do not retry as-is.
- Unknown model returns 400. Verify the model identifier (with or without `@`) before retrying.
- Missing or invalid API key returns 401; insufficient balance or quota returns 402. Top up or switch credentials; do not retry the same call.
- `score` criteria order defines the scale (first entry is the top). Reordering levels changes the meaning of past scores.

## Escalation

- The task needs prose, tools, or conversation: load `references/text-inference/`.
- The task needs grounded classification of retrieved documents: load `references/text-tools/`.
- The decision result looks wrong (bad labels, drifted scores): re-check `instructions` and `criteria` wording first; only then escalate to `references/observability/` with the decision `id` and the exact request payload.
- The failure is a rate limit or transient provider error: load `references/resilience/`.

---
name: aivax-decisions
description: Evaluate named noul, choice, and score questions against a shared state with a semantic decision model. Load when the agent must classify, route, or score a text in one billed call instead of running a chat completion.
---

# Decisions

This sub-skill owns semantic decisions: one `POST` evaluates one or more named questions against a shared JSON `state` and returns structured answers plus usage. It is a classifier/router/scorer, not a chat model — there is no message history, no tools, and no free-text reply.

## When To Use

Use this sub-skill when the agent must evaluate a shared state against one or more fixed criteria in a single call: boolean triage (`noul`), picking one of several named categories (`choice`), or grading intensity on an ordered scale (`score`).

Do not use this sub-skill for:

- Free-text generation, summarization, or multi-turn conversation. Use `references/text-inference/` instead.
- Document classification with grounding evaluation. Use `references/text-tools/` instead.
- Searching or reranking a knowledge base. Use `references/rag/` or `references/rerankers/` instead.

## Endpoints

- **Evaluate Decisions** (`POST /api/v1/generations/decisions`): the only endpoint. RAW envelope (no wrapping). Requires positive balance (`402` otherwise).

Request shape:

```json
{
    "model": "@supersonic-labs/julia-1",
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

Rules: `model` must match a registered identifier exactly, including `@`; discover canonical names and aliases with `GET /api/v1/information/decisions-models.json`, whose `maxImages` tells how many images a model accepts per request (`0` means text-only). `state` is any present JSON value (including `null`, an empty string, object, or array) evaluated by every question.

Image-capable models (`maxImages` above `0`) also take images in a `state` that is a top-level array, mixed with plain-string text items: `{ "type": "image_url", "image_url": { "url": "data:image/png;base64,..." } }`. Only base64 data URLs of `image/png`, `image/jpeg`, or `image/webp` are accepted; remote URLs are never fetched, and images nested inside objects are not read. Images sent to a text-only model, more images than `maxImages`, or any other image URL return 400, so check `maxImages` before attaching images. `questions` is a non-empty object keyed by question name (names cannot be empty). Each question needs non-empty `instructions` plus `criteria`: an object with non-empty `true`/`false` strings for `noul`, a non-empty object mapping choice names to non-empty descriptions for `choice`, or a non-empty ordered array of non-empty strings for `score` (starting at index 0). Any other `type` returns 400.

Illustrative Julia-1 response shape (`id`, `model`, `provider`, `answers` keyed by question name, `usage`; numbers are not a measured result):

```json
{
    "id": "d_example",
    "model": "@supersonic-labs/julia-1",
    "provider": "<provider>",
    "answers": {
        "is_urgent": { "type": "noul", "noul": 0.97 },
        "department": { "type": "choice", "choice": "billing" },
        "frustration": {
            "type": "score",
            "score": 1.1,
            "legend": { "0": "Calm", "1": "Frustrated", "2": "Very angry" },
            "probabilities": { "0": 0.2, "1": 0.5, "2": 0.3 }
        }
    },
    "usage": { "input_tokens": 120, "output_tokens": 0, "cost": 0 }
}
```

Answer fields by type: `noul` carries a numeric result for the true criterion; `choice` carries the selected choice identifier; `score` carries a numeric result and may include a `legend` mapping level indices to descriptions. `probabilities`, `legend`, and `confidence` are optional; never treat absence as zero. For Julia-1, `score` is the expected zero-based level index (the example is `0 × 0.2 + 1 × 0.5 + 2 × 0.3 = 1.1`), not a normalized 0–1 value. Keep the requested level order consistent, and validate scoring semantics when changing models. `usage` carries `input_tokens`, `output_tokens`, and the actual billed `cost`; Julia-1 does not generate text and reports zero output tokens. The API generates a new decision `id`, returns the model's canonical identifier (which may differ from a requested alias), and replaces provider cost with the locally billed amount.

## Cost Awareness

- One request counts once against the account-level request rate limit, regardless of question count. The response `usage.cost` is the billed amount; applicable token usage is metered under `serviceusage.semantic_decisions.in` and `serviceusage.semantic_decisions.out`. A zero cost can reflect included allowance or a zero-priced model; a positive account balance is still required.
- Group related questions when they share a state and fit the selected model's limits, but do not assume batching is cheaper: for Julia-1, input usage sums the encoded sequence for each question, counting the shared state again for each question.
- Prefer decisions over a chat completion when the output is a label, a pick, or a number rather than prose; compare actual usage and cost for your selected model.
- Record the decision `id`, model, and `usage` (tokens + cost) alongside the answers so cost spikes stay attributable. For spike analysis, load `references/cost-monitoring/`.

## Validation

- Confirm every requested question name appears in `answers` and each answer `type` matches the requested `type` before acting on the result.
- Treat a missing `probabilities`/`confidence` as "not supplied", not as zero. Do not threshold on a field that is absent.
- On `score`, use the requested ordered levels and any returned `legend` to interpret the value; do not assume every model reports a normalized score.

## Limits

- Unknown `type` values, missing `state`, missing/empty `model`, `instructions`, or `criteria`, and empty question names or choice names return 400. A present empty `state` is valid. Fix invalid payloads rather than retrying unchanged.
- Unknown model returns 400. Check the exact registered identifier (including `@`) in the decision-model catalog before retrying. Catalog entries change: a model may gain a newer canonical name and keep the old one as an alias, so prefer the returned canonical `name` and re-read the catalog instead of hardcoding identifiers.
- A provider that rejects the request contents (for example an over-long state) is reported as 400 with the provider's message, not as a retryable failure.
- Missing or invalid API key returns 401; insufficient balance or quota returns 402. Top up or switch credentials; do not retry the same call.
- Account-level rate limits are shared across API keys and models: a request above the limit returns 429 before evaluation. Back off rather than switching keys on the same account.
- Model-specific context, question, option, and payload limits differ; consult the current [semantic decision model limits](https://docs.aivax.net/docs/limits.html#semantic-decision-model-limits) rather than applying one model's limits to all. Julia-1 requires 1–32 questions and 2–20 choices or score levels per question; it rejects over-budget inputs and the reserved `<mask>` marker.
- `score` criteria order defines the indexed levels. Reordering levels changes the meaning of past scores.

## Escalation

- The task needs prose, tools, or conversation: load `references/text-inference/`.
- The task needs grounded classification of retrieved documents: load `references/text-tools/`.
- The decision result looks wrong (bad labels, drifted scores): re-check `instructions` and `criteria` wording first; only then escalate to `references/observability/` with the decision `id` and the exact request payload.
- The failure is a rate limit or transient provider error: load `references/resilience/`.

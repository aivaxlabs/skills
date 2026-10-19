# Model Catalog

How to discover and compare AIVAX models for a text-inference task. The catalog is a moving target; treat the live call as the source of truth and this file as the decision shape.

## Discovery

MCP:

```text
aivax_list_models({ "name_filter": "gpt 5 mini" })
```

Fallback HTTP:

```text
GET https://inference.aivax.net/v1/models
```

The response is OpenAI-compatible: `{ "object": "list", "data": [ ... ] }`. Each entry has `id`, `object`, `created`, and `owned_by`. Gateway entries can be used as the `model` value in chat completions.

Use `GET /v1/models?filter=<URL-encoded-model-name>` for a model family and its dated snapshots, ordered by closest match and then newest snapshot. This is not a substring search: matching ignores the vendor prefix and normalizes name separators, so inspect the returned IDs rather than assuming the filter pins a vendor.

For integrated models, discover provider tags in `_details.provider_model.providers[].tag`. Use these exact tags in `routing_options.allowed_providers` or gateway `parameters.allowedProviders`; do not invent tags from vendor display names. In `GET /api/v1/information/models.json` the routing tag is `routingTag`; its `tag` field is the raw provider tag and can differ (for example `/` instead of `-`). Provider order limits the candidate pool before the routing preset chooses within it.

## Decision Fields

For every candidate model, capture the following decision axes before recommending it. These are comparison labels, not guaranteed JSON field names: MCP and HTTP catalogs differ. Read the returned capabilities and provider metadata, and verify unfamiliar fields with `aivax_search_context`.

| Field | Why It Matters |
| --- | --- |
| `provider` and `name` | Whether the model is integrated (consumes AIVAX balance) or BYOK (consumes the provider key). |
| `context_window` | Maximum prompt+completion tokens. Plan context caps may be lower. |
| `max_output_tokens` | Maximum completion length. |
| `modality` | Text, image, audio, video, file. Mismatched modality fails or preprocesses. |
| `tools` | Whether the model supports function calling and which tool surface. |
| `structured_output` | Whether `response_format` / `response_schema` is supported. |
| `reasoning` | Whether the model has a reasoning phase; affects latency and cost. |
| `speed` | Latency class. Real-time workloads need a fast class. |
| `intelligence` | Quality class. Hard reasoning needs a high class. |
| `pricing` | Input, output, cached input, audio input prices per million tokens. |
| `plan_availability` | Whether the model is available on Free, Pro, Max. |
| `flags` | Preview, deprecated, unstable, regional, BYOK-only. |
| `cache` | Whether the model supports cached input (lower cost on repeated prefixes). |

## Compare Two Candidates

Use this skeleton when the agent must recommend one model over another.

```text
candidate_a:
  name: <name>
  context_window: <tokens>
  modality: <list>
  pricing:
    input_per_mtok: <usd>
    output_per_mtok: <usd>
    cached_input_per_mtok: <usd>
  speed: <class>
  intelligence: <class>
  tools: <bool>
  structured_output: <bool>
  reasoning: <bool>
  plan_availability: <list>
  flags: <list>
candidate_b:
  <same shape>
decision:
  recommended: <a|b>
  rationale: <one or two lines>
  expected_cost_delta: <estimate>
  expected_quality_delta: <estimate>
```

## Pricing Rules Of Thumb

- Consult [Pricing](https://docs.aivax.net/docs/pricing) and [Limits](https://docs.aivax.net/docs/limits); do not assume fixed input, output, cache, or reasoning price ratios.
- Reuse stable prefixes when the selected provider supports input caching.
- Compare the actual input, output, audio, and reasoning usage for the workload before recommending a swap.

## When To Recommend A Swap

Recommend a model swap when:

- The current model is consistently more expensive than a cheaper model with the same capability floor.
- The current model is too slow for the workload (real-time, batch) and a faster model is available with acceptable quality loss.
- The current model lacks a required capability (modality, tool surface, structured output, context window).
- A deprecated or preview flag was raised and a stable alternative exists.

Do not recommend a swap when:

- The current model is the only one with the required capability.
- The quality delta is unknown and the workload is production.
- The expected saving does not justify retraining, retesting, or rollback risk.

## Caveats

- `aivax_list_models` is a live call. Do not cache the result across sessions; the catalog changes.
- Verify that the account plan permits the selected model and workload; a listed ID alone does not establish eligibility.
- Gateway entries in the model list are usable as `model` in chat completions, but they are not the model itself; the gateway's model, instructions, and skills apply.

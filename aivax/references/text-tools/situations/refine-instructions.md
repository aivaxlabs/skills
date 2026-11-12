# Refine Instructions

Use when the agent must review LLM context — system instructions, prompts, skills, tool descriptions, knowledge, or documentation — against prompt-engineering guidelines through `POST /api/v1/generations/instruction-refinement`, and get located findings plus fixes as unified diff patches.

## Objective

Get a scored review of the context one agent receives, with prioritized findings pointing at exact lines and fixes that can be previewed or applied as patches, without building a chat completion.

## Preconditions

- A key with the `InferenceGenerations` permission; call it server-side with a private key (`sk-aiv-acc`) when the context is confidential.
- A positive balance; otherwise the call returns `402`.
- The full text of every item, exactly as the agent receives it. Findings and patches refer to the `content` you send, so do not send a summarized or reformatted copy.

## Decision Tree

1. Wording of each item (clarity, constraints, examples, format)? Use `view: "micro"` (the default). How the items work together as one agent (purpose, authority, placement, skill activation, capabilities)? Use `view: "macro"`.
2. Send every item the same agent receives in one `context` array; conflicts and duplications are only found between items reviewed together. Use the matching `type`: `instruction` (always-loaded system or developer text), `prompt` (user-turn prompt), `skill` (on-demand bundle; put its activation text in `description`), `tool`, `knowledge`, or `documentation` (notes for humans the agent never sees).
3. State the agent's goal in `purpose` when it is known; the review judges whether the context serves it.
4. Give each item a `title` (file or skill name). It becomes the path in the patch headers; untitled items appear as `context-<index>`.
5. Large or repeated reviews: every `context` item counts against the document-processing quota shared with extraction and summarization.

## Construction

```json
POST /api/v1/generations/instruction-refinement
{
  "view": "macro",
  "purpose": "Answer billing questions and hand refunds to the finance team.",
  "context": [
    { "type": "instruction", "title": "support-agent.md", "content": "You are a support agent.\nNever promise refunds." },
    { "type": "skill", "title": "refunds", "description": "Refunds.", "content": "Promise a refund when the customer asks for one." }
  ]
}
```

Only `view`, `purpose`, and `context[]` with `type`, `title`, `description`, and `content` are read. There is no `model` field.

## Response Handling

The standard `{ message, data }` envelope. `data` contains:

- `overallScore`: 0 to 1, lower when higher-priority findings apply; `summary`: a short overall assessment. Text fields follow the language of most of the analyzed content.
- `findings[]`: `id` (from 1), `priority` (`P0` to `P3`, `P0` most severe), `title`, `description`, `locations[]`, and `sources[]`.
  - Each location has `contextIndex` (position in the request `context`) and `evidenceSpan` `{ fromLine, toLine, fromColumn, toColumn }`, 1-based and inclusive. `evidenceSpan` is `null` when the problem is an omission, lies in a skill `description`, or the quoted text could not be located.
  - Each source has the guideline `id` (such as `A.R. 2` in micro or `B.D. 3` in macro), `number`, `type` (`rule`, `practice`, `tip`, or `dont`), and `references` (vendor documentation URLs, possibly empty).
- `suggestedFixes[]`: `refFinding` (finding ids it resolves), `type` (`modify`, `remove`, `transfer`, or `add`), `description` (what to change and where), and `patches[]` with `contextIndex` and `diff`.

Handle patches as follows:

1. `diff` is a unified diff of one item: `--- a/<title>` and `+++ b/<title>` headers, `@@` hunks with 3 lines of context, and `\ No newline at end of file` when the item does not end with a line break. The item's line-ending style is preserved.
2. Every patch is computed against the original `content` you sent, independently of other fixes. After applying one fix, re-run the review or re-apply the next patch with fuzz; two fixes can touch the same lines.
3. An empty `patches` array means the fix is described but not applied as text: it needs facts only the author has (a limit, a contact, a policy), needs a new item or tool, or its edits did not match the item exactly once. Read `description` and write the change manually; do not invent the missing facts.
4. Review each patch before applying it. Fixes are model output; they can change behavior the author intended.

## Billing

Billed per request in processing units from the token usage of the model that served it, recorded as `@aivax/instruction-refinement` (`serviceusage.instruction_refinement.pu`) and attributed to the API key. Read amounts from usage records and [Pricing](https://docs.aivax.net/docs/pricing), not from this skill. Larger and more numerous items cost more.

## Failure Modes

- `400`: `view` is not `micro` or `macro`; `context` is missing or empty; an item has an unsupported `type` or empty `content`. Fix the payload; do not retry unchanged.
- `401`: missing or invalid key. `403`: the key lacks the `InferenceGenerations` permission.
- `402`: no positive balance or the storage quota is exceeded.
- `429`: the document-processing limit was exceeded. Load `references/resilience/situations/rate-limit-429.md`.
- `503`: the service is unavailable or returned an invalid review. Retry later with backoff.

## Escalation

- The reviewed context is an AIVAX account skill or gateway instruction: apply the reviewed changes with `references/skill-development/` or `references/ai-gateways/`.
- The agent's behavior must be verified after the fixes: load `references/agentic-tests/`.
- The task needs a free-form critique or rewrite instead of guideline findings: load `references/text-inference/`.

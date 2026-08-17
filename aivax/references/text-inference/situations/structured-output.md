# Structured Output

Use when the agent must produce or consume JSON from a chat completion, enforce a schema, or recover from a malformed model output.

## Objective

Get valid JSON out of the model, on a known schema, with the smallest retry surface, and without paying for a full regeneration when the only problem is a missing field.

## Preconditions

- A JSON Schema for the desired output. The schema names every required field, types, and any constraints (enums, ranges, formats).
- A model that supports structured output (verify with `aivax_list_models`).

## Decision Tree

1. Is the output intended for a downstream system that already expects a `response_format` envelope? Use `response_format: { type: "json_schema", json_schema: { ... } }`. This is the provider-compatible path.
2. Is the output a tool-friendly JSON for the model itself, and JSON Healing is acceptable? Use `response_schema: { ... }`. AIVAX validates, runs JSON Healing if needed, and returns the validated JSON.
3. Does the caller want only the generated JSON, not the chat-completion envelope? Add `json_only: true`.
4. None of the above? Use a plain `messages` call with a system message that names the schema. This is the most fragile path and should be the last resort.

## JSON Healing

JSON Healing is AIVAX's automatic correction loop. When the model produces invalid JSON, AIVAX asks the model for JSON, extracts it from text or markdown blocks, validates it against the schema, and retries with feedback until the output is valid or the attempt limit is reached.

- JSON Healing is enabled by `response_schema` and optionally by `response_format` with `json_schema` when account settings allow it.
- JSON Healing preserves the model's reasoning phase in reasoning models.
- JSON Healing has a cost: each retry consumes tokens. The agent should not rely on it for every call; a good schema and a clear system message reduce the retry rate.

## Schema Design

- Use `additionalProperties: false` to keep the model from inventing extra fields.
- Use `required` to make critical fields non-negotiable.
- Use `enum` for closed sets. The model respects enums better than free-form descriptions.
- Use `description` to give the model context for each field. A short sentence is enough.
- Use `format` for known string formats (date-time, email, uri). The model emits values that match the format more often.

## Validation

- The output is a valid JSON object.
- The output validates against the JSON Schema.
- The required fields are non-null.
- The enums and formats are respected.
- The chat-completion envelope is present (or absent when `json_only: true`).

## Failure Modes

- The model emits valid JSON that does not match the schema: this is a schema design problem. Tighten the schema or clarify the description.
- The model emits JSON wrapped in a markdown code block: AIVAX usually extracts it, but if not, add a system message: "Return only the JSON object. Do not wrap it in markdown."
- The model emits extra prose around the JSON: same remedy.
- The model emits an empty object: usually a token limit. Lower the prompt size or switch to a larger context window.
- JSON Healing retries exceed the attempt limit: surface the partial JSON and the last validation error to the user. The retry budget was the cost cap.

## Escalation

- The schema is so complex that the model cannot produce valid JSON: split it into two calls (one per schema) and compose the result.
- The cost of JSON Healing is unacceptable: switch from `response_schema` to a hand-written parser over plain text, and only fall back to `response_schema` when the parser fails too often.
- The validation reveals the model is hallucinating fields not in the schema: tighten `additionalProperties: false` and reduce the prompt.

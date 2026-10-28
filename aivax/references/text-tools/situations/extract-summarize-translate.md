# Extract, Summarize, Or Translate Documents

Use when the agent must turn documents into schema-shaped JSON (`POST /api/v1/generations/extract`), condense them (`POST /api/v1/generations/summarize`), or translate them (`POST /api/v1/generations/translate`) without building a chat completion.

## Objective

Get one result per document, in input order, from a fixed AIVAX pipeline with no model selection.

## Preconditions

- A private key (`sk-aiv-acc`). These routes are not on the public-key list.
- A positive balance; otherwise the call returns `402`.
- Documents are OpenAI-compatible content parts. Extraction and summarization accept `text`, `image_url`, and `file`; images and files are base64 data URLs or HTTP(S) URLs of JPEG, PNG, TIFF, or PDF content up to 5 MB. Translation accepts `text`, `image_url`, and `input_audio`; images are JPEG, PNG, WebP, or GIF up to 5 MB, and audio is base64 data.

## Decision Tree

1. Need named fields as JSON? Extract. Need a condensed text? Summarize. Need another language? Translate.
2. Extraction: describe every field as a schema property; the property `description` is the search criterion. Nested objects and arrays of objects work; `$ref` and `$defs` do not.
3. Summarization: put format, language, focus, or length in `instructions`. The summary does not add information or commentary.
4. Translation: `toLanguage` is free text (for example `"pt-BR"` or `"Japanese (formal)"`). Omit `fromLanguage` to detect it. Images are translated from their visible text and audio from its speech.
5. Large job? Split it; extraction and summarization count every document against the document-processing quota, translation counts requests. Load `references/batch/` for very large sets.

## Construction

```json
POST /api/v1/generations/extract
{
  "instructions": "Only consider the issuing company as the vendor.",
  "responseSchema": {
    "type": "object",
    "properties": {
      "invoice_number": { "type": "string", "description": "invoice number" },
      "total": { "type": "number", "description": "total amount due" }
    }
  },
  "documents": [
    { "type": "file", "file": { "filename": "invoice.pdf", "file_data": "https://example.com/invoice.pdf" } }
  ]
}
```

```json
POST /api/v1/generations/translate
{
  "toLanguage": "Brazilian Portuguese",
  "documents": [ { "type": "text", "text": "Your order has shipped." } ]
}
```

`/summarize` takes `documents` and optional `instructions`.

## Response Handling

1. The standard `{ message, data }` envelope; `data` has one entry per document, in input order.
2. Extraction: one object shaped by the schema. Fields not found are `null`; values are converted to the schema type when possible and kept as strings otherwise.
3. Summarization and translation: one string per document. A translation can be empty when there is nothing to translate.
4. Translation uses temperature 0 and low reasoning settings, but it is still model output; spot-check it.

## Billing

- Extraction and summarization are billed on input tokens per document (usage categories `serviceusage.document_extraction.in` and `serviceusage.document_summarization.in`). Translation is billed per document in processing units from the token usage of the model that served it, recorded as `@aivax/translation` (`serviceusage.translation.pu`). Read amounts from usage records and [Pricing](https://docs.aivax.net/docs/pricing), not from this skill.
- Documents are processed in parallel and each successful one is billed. One failing document fails the whole request, so documents that already succeeded can be billed without being returned.

## Failure Modes

- `400`: missing or invalid `responseSchema`, `toLanguage`, or `documents`; an unsupported content type; or a document the service could not read.
- `429`: the document-processing or translation limit was exceeded. Load `references/resilience/situations/rate-limit-429.md`.
- `503`: the service is temporarily unavailable. Retry later with backoff.

## Escalation

- The task needs a model choice, tools, or a conversation: load `references/text-inference/`.
- The extraction must follow a strict chat-completion JSON contract instead: load `references/text-inference/situations/structured-output.md`.
- Only a description of the media is needed: load `situations/describe-media.md`.

---
name: aivax-web
description: Search the web and extract readable text from URLs or inline files via direct AIVAX API calls, with optional schema-guided JSON conversion. Load when the agent must call POST /api/v1/web/search or POST /api/v1/web/fetch directly.
---

# Web

This sub-skill owns the direct web search and content extraction endpoints: `system.v1.web.search` and `system.v1.web.fetch`. These are explicit API calls the agent makes itself, not model-driven browsing inside a chat completion.

All routes share the prefix `/api/v1/web`.

## When To Use

Use this sub-skill when the agent must search the web for current information with explicit query parameters, or extract readable text from known URLs or inline base64 files, optionally converting each item's text into structured JSON.

Do not use this sub-skill for:

- Model-driven browsing inside `/v1/chat/completions` (`builtin_tools` such as `WebSearch` or `OpenUrl`). That belongs to text inference; load `references/text-inference/` instead. Rule of thumb: if the model decides when to browse during a completion, it is text-inference; if the agent calls search or fetch directly with its own parameters, it is this sub-skill.
- Searching a RAG knowledge base or collection. Use `references/rag/` instead.
- Classifying, segmenting, or describing already-held text or media. Use `references/text-tools/` instead.

## Endpoints

- **Search Web** (`POST /api/v1/web/search`, `system.v1.web.search`): searches the web with optional country, language, and domain filters. Returns `results` items with `url`, `title`, and `text`.
- **Fetch Web Contents** (`POST /api/v1/web/fetch`, `system.v1.web.fetch`): batch extraction over `contents` (public URLs or base64 data URIs). Returns `results` items with the zero-based input `index`, `extractedText`, `extractedObject`, `processingUnits`, `jsonProcessingUnits`, and `error`.

## Request / Response

Search request:

```json
{
    "query": "latest artificial intelligence research",
    "topn": 5,
    "country": "US",
    "language": "en",
    "includeDomains": ["arxiv.org", "nature.com"]
}
```

- `query` is required. `topn` defaults to 10 when omitted. `country`, `language`, and `includeDomains` are optional filters.

Fetch request (text-only):

```json
{
    "contents": [
        "https://example.com/article",
        "data:text/plain;base64,SGVsbG8gd29ybGQ="
    ],
    "returnErrors": true
}
```

Fetch request with structured extraction:

```json
{
    "contents": ["https://example.com/files/receipt.png"],
    "returnErrors": true,
    "responseSchema": {
        "type": "object",
        "instructions": "Extract only information present in the receipt. Use null for missing values.",
        "properties": {
            "merchant": { "type": ["string", "null"] },
            "date": { "type": ["string", "null"] },
            "total": { "type": ["number", "null"] }
        },
        "required": ["merchant", "date", "total"],
        "additionalProperties": false
    }
}
```

- `contents` is required and non-empty; each entry is a URL or a base64 data URI of the form `data:<mime-type>;base64,<payload>`. The data URI must specify a MIME type and base64 content.
- `returnErrors` defaults to `true`: each failed item returns an entry with null extracted values, zero PU fields, and an `error` message. When `false`, failed items are omitted from `results`.
- `responseSchema` is optional; omit it or set it to null for text-only extraction. `instructions` inside the schema is an AIVAX extension (not a standard JSON Schema keyword) carrying extraction guidance. Successful JSON conversion retains `extractedText`; without a schema, `extractedObject` is null and `jsonProcessingUnits` is 0.
- Match each result to its input via the zero-based `index`. User-visible failures (invalid input, fetch or conversion errors) surface their message in `error`; unexpected failures surface a generic fetch failure message.

## Cost Awareness

- Search bills one web-search charge per request, independent of result count.
- Fetch bills text/OCR extraction per item through its `processingUnits` field.
- JSON conversion bills separately per converted item through `jsonProcessingUnits` (usage category `serviceusage.ocr.json_post_process`). It is inference-based, reported apart from extraction PUs, and sits outside the text/OCR daily allowance. Only send `responseSchema` when structured output is actually needed.
- Both endpoints require a positive balance (402 otherwise).

## Validation

- For search, confirm `results` covers the query before using any item; an empty result set is a valid answer, not a retry trigger.
- For fetch, confirm each expected `index` is present (or, with `returnErrors: false`, account for omitted failures) before acting on the batch.
- With `responseSchema`, validate `extractedObject` against the schema; a conversion failure arrives as an `error` item, not partial JSON.
- The trace ID is preserved.

## Limits

- Each fetch item has a maximum content size, applied to fetched URL content and decoded data-URI bytes alike. Do not rely on a remembered number (the endpoint's own description text can lag behind the enforced size); the error message states the enforced limit. Oversize items fail with an error entry (or omission when `returnErrors` is false). See Plans and Limits for plan-level constraints.
- Large scanned documents are converted page by page and can take minutes: keep batches small, set a generous client timeout, and retry only the failed `index` entries rather than the whole batch.
- Search: `topn` must be between 1 and 25; `country` must be a two-letter ISO 3166-1 code; `language` must be a valid language code such as `pt` or `pt-BR`; `includeDomains` cannot contain empty values.
- Fetch: `contents` must contain at least one item; each item must be non-empty.
- Rate limits apply per account (429 on excess); search carries an additional web-search-specific limit.

## Escalation

- The model should decide when to browse inside a completion: load `references/text-inference/` and use its built-in tools instead.
- The same query needs retry, backoff, or partial-batch recovery policy: load `references/resilience/`.
- Spend attribution or allowance questions arise: load `references/cost-monitoring/`.
- A failure needs cross-request diagnosis: load `references/observability/`.

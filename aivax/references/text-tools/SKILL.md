---
name: aivax-text-tools
description: Use AIVAX text tools — document classification, text segmentation, and media description — to prepare, structure, or summarize content before or instead of chat completion. Load when the agent needs to classify documents, segment long text into chunks, or generate a textual description of media.
---

# Text Tools

This sub-skill owns AIVAX's non-chat text tools: document classification, text segmentation, and media description. These tools are designed for the pre- and post-processing stages of a pipeline: classify an incoming document, segment a long text, or describe a media file so a downstream model can reason over the description.

## Operating Files

- `situations/classify-documents.md`: classify one or many documents.
- `situations/segment-text.md`: segment a long text into chunks.
- `situations/describe-media.md`: produce a textual description of a media file.

## Endpoints

- Classification: `POST /api/v1/generations/classify`.
- Segmentation: `POST /api/v1/generations/segment`.
- Media description: `POST /api/v1/generations/descriptions`.

None of these endpoints accepts a `model` field. Their request bodies and response shapes are distinct:

- Classification accepts `documents` and `labels`; the response envelope contains `{ "results": [...] }` in `data` (`OpenIndexer.Alpha2/Api/Modules/Generations/ClassificationsApi.cs:39-64,74-81`).
- Segmentation accepts `documents` and optional `sanitize`; the response envelope contains `{ "result": [...], "usage": {...} }` in `data` (`OpenIndexer.Alpha2/Api/Modules/Generations/SegmenterApi.cs:57-61,154-160`).
- Media description accepts `input`, optional `extractionGuidance`, and optional `preset`; `preset` is `auto` (default) or `high`. The response envelope's `data` is an array with one resolver-generated object per input item (`OpenIndexer.Alpha2/Api/Modules/Generations/MediaDescriptionsApi.cs:117-121,174`).

## When To Use Text Tools

Use this sub-skill when the agent needs to:

- Classify a document by intent, topic, language, urgency, or another label.
- Segment a long text into chunks for downstream indexing, retrieval, or display.
- Produce a textual description of a media file (image, audio, video, file) that a downstream model can reason over.

Do not use this sub-skill for:

- Chat completion (load `references/text-inference/`).
- Multimodal input to a chat completion (load `references/multimodal/`).
- RAG retrieval (load `references/rag/`).
- Reranking (load `references/rerankers/`).

## Quotas

Text-classification and text-segmentation quotas each count every item in the request's `documents` array, not each HTTP request. A request that sends 100 documents consumes 100 quota items. Plan accordingly; for large jobs, load `references/batch/`.

## Cost Awareness

These tools are priced per item, not per token. A request that sends 1,000 documents can cost more than a long chat completion. Set a cost cap with the user before running large jobs.

## Validation

- Classification returns `data.results`, with one indexed result per document and every label ordered by score.
- Segmentation returns `data.result`, with indexed segments per document, plus `data.usage`.
- Media description returns an array directly in `data`, preserving input order.

## Limits

- Per-request item count limits exist. Verify them with `aivax_search_context`.
- Language support varies by tool. Verify before sending multilingual content.
- Per-account quotas exist. Large jobs can hit the quota. Load `references/resilience/situations/rate-limit-429.md` when they do.

## Escalation

- The classification is wrong: clarify the labels or clean noisy documents.
- The segmentation is wrong: adjust `sanitize` or the source document structure.
- The description is wrong: adjust `extractionGuidance` or `preset`, or improve the input quality.
- The task is large: load `references/batch/`.

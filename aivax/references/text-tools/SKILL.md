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

- Classification: `POST /api/v1/generations/classify-documents`.
- Segmentation: `POST /api/v1/generations/segment-text`.
- Media description: `POST /api/v1/generations/describe-media`.

All three accept a list of items and return a list of results. The model and parameters are part of the request body. Verify the model and parameter shape with `aivax_search_context` before relying on a field.

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

## Models

Verify with `aivax_list_models`. Each tool has a catalog of supported models. The choice of model affects cost, quality, and language coverage.

## Cost Awareness

These tools are priced per item, not per token. A request that sends 1,000 documents can cost more than a long chat completion. Set a cost cap with the user before running large jobs.

## Validation

- The response is a list of results, one per input item.
- The results match the input order (when the model preserves order; verify with `aivax_search_context`).
- The trace ID is preserved through `metadata`.

## Limits

- Per-request item count limits exist. Verify with `aivax_search_context` for the model.
- Some models support only a subset of languages. Verify before sending multilingual content.
- Per-account quotas exist. Large jobs can hit the quota. Load `references/resilience/situations/rate-limit-429.md` when they do.

## Escalation

- The classification is wrong: the labels are ambiguous, the model is not strong on the domain, or the input is noisy. Adjust the labels or switch the model.
- The segmentation is wrong: the chunk size is too small or too large, the model is not strong on the language, or the input is structured in a way the model does not understand. Adjust the parameters or switch the model.
- The description is wrong: the model is not strong on the modality, the input is too noisy, or the prompt is too vague. Adjust the parameters or switch the model.
- The task is large: load `references/batch/`.

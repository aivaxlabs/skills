# Segment Text

Use when the agent must split long source text into retrieval-ready strings through `POST /api/v1/generations/segment`. This returns segments; it does not create collection documents or index them.

## Objective

Choose whether omission is acceptable, inspect segment quality, and pass only reviewed segments to the downstream task.

## Preconditions

- Supply a non-empty `documents` array of non-blank strings.
- Preserve the original source for comparison and keep input indexes associated with source IDs in the application.

## Decision Tree

1. Is each source already focused and self-contained? Skip segmentation and ingest directly.
2. Must all source content remain represented? Use `sanitize: false` (the default). It follows structural and semantic boundaries in source order without sliding overlap; do not treat the output as a byte-for-byte copy because whitespace and formatting can be normalized.
3. Is removing irrelevant material acceptable? Use `sanitize: true` only then. It uses a language model and may omit frontmatter, references, or passages unsuitable for standalone retrieval; review the omissions.
4. Are custom boundaries, overlap, or exact formatting required? Prepare the chunks in the application. The endpoint does not accept a model, chunk-size, overlap, language, or structure parameter.

## Construction

```text
POST /api/v1/generations/segment
{
  "documents": ["<complete source text>", "<another source text>"],
  "sanitize": false
}
```

Do not send `model`, `text`, `max_chunk_size`, `overlap`, or `metadata` as segmentation controls. Keep correlation metadata in the application and add document metadata when ingesting the returned segments.

## Response Handling

1. Read `data.result`; each item has `index`, `count`, and `segments` (strings). Map results by `index`, not response-array position: sanitized results can arrive out of input order.
2. Inspect every item for `error`, including on successful HTTP responses. A sanitized item failure carries `error`, an empty `segments` array, and `count: 0`; an empty result without `error` also needs review before ingestion.
3. Inspect `data.usage.processing_units` and `data.usage.cost`. Do not expect token counters, character offsets, or echoed metadata.
4. Review coverage and boundaries, then upsert or import the segments into the collection with stable names and source references. Wait for indexing before testing retrieval.

## Validation

- Every source index has a reviewed result; retain failed items separately for deliberate retry.
- Without sanitization, compare the segments with the source for content coverage, allowing normalization and repeated structural headers where needed.
- With sanitization, confirm omitted passages are intentionally excluded; never promise lossless output.
- Inspect headings, tables, code blocks, and neighboring segments for enough context to stand alone.
- Keep source IDs and request correlation outside the segmentation payload.

## Failure Modes

- HTTP `400`: inspect malformed fields, an empty array, blank documents, or the returned processing failure. Do not retry invalid input unchanged.
- HTTP `429`: the per-document segmentation quota was reached. Load `references/resilience/situations/rate-limit-429.md`.
- An item has `error` or no useful segments: inspect that item before retrying; do not ingest it as an empty document.
- Boundaries are unsuitable: clean the source structure or prepare chunks yourself; there is no chunk-size knob or selectable model on this endpoint.

## Escalation

- The segments are for RAG: load `references/rag/situations/design-rag-pipeline.md`.
- The segments are for classification: load `situations/classify-documents.md`.
- A source cannot be processed in one request: split it at meaningful boundaries in the application and preserve the source-to-segment mapping.

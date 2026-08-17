# Segment Text

Use when the agent must segment a long text into chunks through `POST /api/v1/generations/segment-text`. Common uses: preparing text for RAG indexing, splitting a long document for display, or chunking for downstream classification.

## Objective

Split a long text into meaningful chunks, with the right chunk size, overlap, and structural awareness.

## Preconditions

- A segmentation model is available on the account. Verify with `aivax_list_models`.
- The text is in a supported language.

## Decision Tree

1. What is the purpose of the segmentation? RAG indexing, display, or downstream classification. Each has different chunk-size sweet spots.
2. What is the chunk size? A few hundred tokens is a good default for RAG; larger is fine for display; smaller is fine for classification.
3. Is overlap needed? Some overlap helps RAG retrieval when a fact spans a chunk boundary.
4. Should segmentation respect structure (headings, paragraphs, code blocks)? Some models do this natively; others split on token count alone.
5. Is the text structured (Markdown, HTML, code)? Use a model that respects the structure. Verify with `aivax_search_context`.

## Construction

```text
{
  "model": "<segmentation-model>",
  "text": "<long-text>",
  "max_chunk_size": 800,
  "overlap": 100,
  "metadata": { "trace_id": "tr_..." }
}
```

Some models accept `language`, `respect_structure`, and `unit` (token or character). Verify with `aivax_search_context`.

## Response Handling

1. The response is a list of chunks, in order.
2. Each chunk has the text, the start and end positions (when supported), and any metadata the model returns.
3. Use the chunks for the downstream task: indexing into a RAG collection, sending to a classifier, or displaying to the user.
4. Record the chunk count and the chunk-size distribution in the trace.

## Validation

- The chunks cover the entire input. No text is dropped or duplicated.
- The chunk-size distribution is within the expected range.
- The chunks respect structure (when configured to do so).
- The trace ID is preserved.

## Failure Modes

- The model returns no chunks: the input is empty or unsupported. Inspect the error.
- The chunks are too small or too large: the parameters are wrong. Adjust.
- The chunks break in the middle of a sentence or a code block: the model does not respect structure. Switch the model or set the structural parameter.

## Escalation

- The chunks are for RAG indexing: load `references/rag/situations/design-rag-pipeline.md`.
- The chunks are for classification: load `situations/classify-documents.md`.
- The text is too large for a single request: split the input first and segment each part. Or load `references/batch/`.

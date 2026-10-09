# File Input

Use when the user wants the model to reason about a file (PDF, DOCX, code, structured data, generic binary) by attaching it to a chat completion.

## Objective

Send file content to the model in a way that preserves structure, fits the model's context and modality limits, and respects balance minimums.

## Preconditions

- The file is in a supported format. PDF and common text formats are supported; some formats require preprocessing.
- The request meets the current file requirements in [Limits](https://docs.aivax.net/docs/limits); consult [Pricing](https://docs.aivax.net/docs/pricing) for extraction and inference charges.
- The file is reachable (URL) or can be base64-encoded (data URL).

## Decision Tree

1. Is the file a PDF? Use `multimodal_resolver.fileEngine: "InferenceLow"` or `"InferenceHigh"` for a description, or `"Ocr"` for text extraction. Leave it null only when the main model can consume the file directly.
2. Is it a non-PDF file? Any enabled `fileEngine` uses the OCR/text-conversion path. Confirm format support rather than assuming arbitrary binary files can be extracted.
3. Is extraction unsupported or lossy? Convert to a supported PDF or text representation first, preserving the structure needed for the task.
4. Is the file very large? Segment it before sending. Load `references/text-tools/situations/segment-text.md` for guidance.
5. Is the file a structured dataset (JSONL, CSV)? Use the appropriate structured input mode: `file.file_data` with a data URL is fine for small datasets; for large datasets, load the data into a RAG collection and let the model retrieve from it.

## Construction

```text
{
  "role": "user",
  "content": [
    { "type": "text", "text": "<prompt>" },
    { "type": "file", "file": { "filename": "<name>", "file_data": "<url or data:>" } }
  ]
}
```

For text-only models, add `"multimodal_resolver": { "fileEngine": "InferenceLow" }` to the request, or choose `"Ocr"` for extraction. See `situations/multimodal-preprocess.md` before overriding a gateway resolver.

## Validation

- The model references the file in its response.
- The `usage` block accounts for the file tokens (and any preprocessing tokens).
- The trace ID is preserved.
- The model's response reflects the file's content faithfully.

## Failure Modes

- Unsupported format: convert to a supported format (PDF, TXT, MD) before sending.
- File too large: segment first, then send each segment in a separate message or batch item.
- Preprocessing produces a poor description: load `references/text-tools/situations/describe-media.md` for a more specific model.
- Model returns a refusal: moderation is blocking. Adjust the prompt or the input.

## Escalation

- The file is a large dataset: load `references/rag/` and index it into a collection.
- The file is a recurring input across many requests: index it once and load `references/rag/situations/design-rag-pipeline.md`.
- The task is repetitive across many files: load `references/batch/situations/design-batch-workflow.md`.

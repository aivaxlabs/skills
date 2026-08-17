# File Input

Use when the user wants the model to reason about a file (PDF, DOCX, code, structured data, generic binary) by attaching it to a chat completion.

## Objective

Send file content to the model in a way that preserves structure, fits the model's context and modality limits, and respects balance minimums.

## Preconditions

- The file is in a supported format. PDF and common text formats are supported; some formats require preprocessing.
- The account balance is at least $0.50 (the minimum for file input).
- The file is reachable (URL) or can be base64-encoded (data URL).

## Decision Tree

1. Is the file a PDF? AIVAX preprocesses PDF files with auxiliary multimodal inference when the flag is set. Send directly via `file` and set `multimodal_preprocess: "File"` if the main model is text-only.
2. Is the file a supported text format (TXT, MD, CSV, JSON, code)? Local text extraction is used. Send directly or preprocess as needed.
3. Is the file an unsupported format (DOCX, XLSX, PPTX, binary)? Preprocess with `multimodal_preprocess: "OtherFiles"` to extract what can be extracted. If the extraction is lossy, convert the file to PDF or text first.
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

For text-only models, set `multimodal_preprocess: "File"` (or `"All"`) on the request so AIVAX converts the file to a text description before the main call.

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

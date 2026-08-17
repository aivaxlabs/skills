# Image Input

Use when the user wants the model to reason about one or more images, and the model's modality list includes image input.

## Objective

Send image content to the model in a way that preserves fidelity, fits the model's context and modality limits, and respects balance minimums.

## Preconditions

- The model supports image input. Verify with `aivax_list_models`.
- The account balance is at least $0.10 (the minimum for image input).
- The image is reachable (URL) or can be base64-encoded (data URL).

## Decision Tree

1. Is the image reachable by URL without authentication? Use `image_url.url` with the URL. Faster, smaller payload.
2. Is the image private or behind authentication? Base64-encode the bytes and use a data URL. Slower, larger payload, but reliable.
3. Is the image larger than 20 MB? Prefer a URL. Some providers reject very large base64 payloads. If the URL is not an option, downscale or compress before encoding.
4. Is the model a vision model with `detail` support? Set `image_url.detail` to `high` for fine-grained tasks (OCR, dense text, small details) and `low` for broad tasks (general classification, presence detection). `auto` lets the model decide.
5. Are there multiple images? Send multiple `image_url` parts in the same `content` array, or in multiple `user` messages. The model treats them as a sequence.

## Construction

```text
{
  "role": "user",
  "content": [
    { "type": "text", "text": "<prompt>" },
    { "type": "image_url", "image_url": { "url": "<url or data:>", "detail": "auto" } }
  ]
}
```

## Validation

- The model references the image in its response.
- The `usage.prompt_tokens` includes image tokens.
- The trace ID is preserved.
- The response is the kind the user asked for (description, classification, extraction, etc.).

## Failure Modes

- The model returns a content error: the image is not reachable, the format is unsupported, or the balance is too low. Inspect the error and the resource.
- The model returns a generic answer: the prompt is too vague. Tighten the prompt or set `detail: high`.
- The model returns a refusal: the moderation policy is blocking. Do not bypass moderation; adjust the prompt or the input.

## Escalation

- The image is too large to send: load `references/text-tools/situations/describe-media.md` and preprocess the image to a text description.
- The model cannot see fine details: increase `detail` to `high` or split the image into crops and send each crop.
- The task is repetitive across many images: load `references/batch/situations/design-batch-workflow.md` and process the images in batch.

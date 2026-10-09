---
name: aivax-multimodal
description: Send and receive multimodal content in AIVAX chat completions — images, audio, video, files — including when to use direct media vs. multimodal preprocessing to text. Load when the user input or the model output includes media, or when the model must reason over attached content.
---

# Multimodal

This sub-skill owns multimodal content in chat completions. The OpenAI-compatible content-part shape is the surface; the discipline is knowing when to send media directly, when to preprocess media to text, and how to handle balance and modality limits.

## Operating Files

- `situations/image-input.md`: send an image as a content part.
- `situations/audio-input.md`: send audio as a content part or transcribe first.
- `situations/video-input.md`: send a video as a content part.
- `situations/file-input.md`: send a document or generic file.
- `situations/multimodal-preprocess.md`: convert media to a text description before the main call.

## Content Parts

AIVAX accepts OpenAI-compatible content parts inside the `messages[].content` field. The supported types are:

| Type | Use | Notes |
| --- | --- | --- |
| `text` | Plain text | Default. |
| `image_url` | Image content | `image_url.url` can be an external URL or a base64 data URL. `image_url.detail` can be `low`, `high`, or `auto` when the model supports it. |
| `video_url` | Video content | `video_url.url` can be an external URL or a base64 data URL. Prefer URLs for large videos. |
| `input_audio` | Audio content | `input_audio.data` is base64 audio data. `input_audio.format` names the format (e.g. `wav`, `mp3`). |
| `file` | File content | `file.filename` names the file. `file.file_data` can be an external URL or a base64 data URL. |

The selected model must support the modality. Otherwise, configure that media type in `multimodal_resolver` to convert it into text before the main call.

## External Links

External links must be reachable by AIVAX without authentication, firewall restrictions, or JavaScript-only rendering. Failed downloads, redirects, blocked URLs, unsupported formats, or provider-specific size limits can fail the inference. When in doubt, upload the content as base64 inside the content part.

## Balance Minimums

Check the current media requirements in [Limits](https://docs.aivax.net/docs/limits) and charges in [Pricing](https://docs.aivax.net/docs/pricing). A balance gate is not the request price; actual usage depends on the selected path and model.

## Pre-Processing

Use `multimodal_resolver` on a private-key chat-completion request, or `parameters.multimodalResolverParameters` on a gateway. Its camelCase fields are:

- `imageEngine`: `InferenceLow` (alias `Inference`), `InferenceHigh`, or `Ocr`.
- `audioEngine`: `InferenceLow` (alias `Inference`), `InferenceHigh`, or `Stt`.
- `videoEngine`: `InferenceLow` (alias `Inference`) or `InferenceHigh`.
- `fileEngine`: `InferenceLow` (alias `Inference`), `InferenceHigh`, or `Ocr`. Inference engines describe PDFs; other files still use the OCR/text-conversion path. `Ocr` also extracts PDFs.

A null or omitted engine leaves that media type unchanged; it does not auto-detect a fallback. The request object replaces the resolver configuration rather than merging individual engines. Precedence is `multimodal_resolver`, then deprecated `multimodal_preprocess`, then the gateway configuration. At gateway level, `multimodalResolverParameters` supersedes deprecated `enabledMultimodalFeatures`.

Legacy `Image`, `Audio`, `Video`, and `File` flags map to `InferenceLow`; `OtherFiles` maps to file `Ocr`. Prefer explicit engines for new work. Results are cached per account, content-part hash, and engine; do not promise freshness when a URL's content changes. See `situations/multimodal-preprocess.md`.

## When To Use Each Path

- The model supports the modality: prefer direct media when visual, audio, or temporal fidelity matters; compare cost and latency rather than assuming it is cheaper.
- The model does not support the modality: preprocess to text, accepting the extraction cost and possible loss of detail.
- The content is large and the model is text-only: extract or segment it before the main call; preprocessing is not a bypass for upload or context limits.
- The content is small and the model supports it: send directly.

## Cost Awareness

Multimodal calls can be expensive. Compare the cost of a multimodal-capable model against a text-only model + preprocess for the same task. The right answer depends on the model's per-modality pricing; verify with `aivax_list_models` or the model's pricing page.

## Validation

- The model returns a valid response that references the content.
- For preprocess, the model receives a text description that includes the relevant facts.
- The `usage` block accounts for the media tokens.
- The trace ID is preserved.

## Limits

- The Free plan has a context cap. Multimodal content counts against the context.
- Some models support a subset of modalities. Verify with `aivax_list_models`.
- Insufficient balance returns 402.

## Escalation

- The model returns a content error: load `references/platform-rules/error-handling.md` and classify.
- The preprocess produces a poor description: load `references/text-tools/situations/describe-media.md` and consider a more specific model.
- The media is too large to send directly: load `references/text-tools/situations/segment-text.md` to chunk the file before preprocessing.

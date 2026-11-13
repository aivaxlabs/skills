# Describe Media

Use when the agent must produce a textual description of a media file through `POST /api/v1/generations/descriptions`. Common uses: describing an image for a downstream text model, describing an audio for accessibility, describing a video for a transcript.

## Objective

Produce a faithful textual description of the media, with the right engine and guidance for the downstream task.

## Preconditions

- The account balance is positive.
- The media is an OpenAI-compatible content part: `image_url`, `input_audio`, `video_url`, or `file`. Inference engines accept only PDFs as `file`; `fileEngine: "Ocr"` accepts other documents.
- Remote URLs are publicly reachable. Remote files are downloaded up to 5 MB each.

## Decision Tree

1. What is the modality? Pick the engine for that content type in `options`.
2. Does the task need understanding or only text?
   - Understanding (subjects, scenes, tone, music): `InferenceLow`, or `InferenceHigh` when small details matter and the higher cost is acceptable.
   - Only the visible text of images or documents: `Ocr`.
   - Only the spoken words of audio: `Stt`.
3. Does the description need a focus? Write `extractionGuidance` as the question the workflow needs answered. `Ocr` and `Stt` ignore it.
4. Is the result for a chat model in the same request? Prefer multimodal pre-processing: load `references/multimodal/situations/multimodal-preprocess.md`.

## Construction

```json
{
  "options": {
    "imageEngine": "InferenceHigh",
    "audioEngine": "Stt",
    "videoEngine": "InferenceLow",
    "fileEngine": "Ocr"
  },
  "extractionGuidance": "Identify the main subject and visible text.",
  "input": [
    { "type": "image_url", "image_url": { "url": "https://example.com/image.png" } }
  ]
}
```

| Field | Values | Default |
| --- | --- | --- |
| `options.imageEngine` | `InferenceLow`, `InferenceHigh`, `Ocr` | `InferenceLow` |
| `options.audioEngine` | `InferenceLow`, `InferenceHigh`, `Stt` | `InferenceLow` |
| `options.videoEngine` | `InferenceLow`, `InferenceHigh` | `InferenceLow` |
| `options.fileEngine` | `InferenceLow`, `InferenceHigh`, `Ocr` | `InferenceLow` |

- Omit `options` or any engine to use the default.
- `preset` (`auto` or `high`) is deprecated. Never send it with `options`; the request is rejected with `400`.
- There are no `model`, `detail`, `language`, `max_length`, or `focus` fields. Express language and focus through `extractionGuidance`.

## Response Handling

1. `data` holds one JSON object per input item, in input order.
2. Inference engines return a structured object whose shape depends on the content type (for example, `foregroundSubjects` for images and `textContent` plus `sections` for PDFs).
3. `Ocr` and `Stt` return `{ "textContent": "..." }`.

## Validation

- The description is faithful to the media. Spot-check on a sample.
- The output count matches the input count.
- The selected engine matches the task: no `InferenceHigh` where `InferenceLow` or `Ocr` is enough.

## Failure Modes

- `400` with "cannot be used together": remove `preset`.
- `400` on an engine value: the engine is not valid for that content type (for example, `videoEngine: "Ocr"`).
- `400` "not a multimodal content part supported by the resolver model": a non-PDF file with an inference engine. Use `fileEngine: "Ocr"`.
- Details are missing: switch the engine to `InferenceHigh` or refine `extractionGuidance`.
- `402`: the balance is not positive.

## Escalation

- The description is for a downstream model: load `references/multimodal/situations/multimodal-preprocess.md` and consider preprocessing instead.
- The description is for accessibility: load `references/web-chat/` or another user-facing channel and integrate the description.
- The task is to describe many media files: load `references/batch/`.

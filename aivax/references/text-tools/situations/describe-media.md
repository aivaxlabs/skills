# Describe Media

Use when the agent must produce a textual description of a media file through `POST /api/v1/generations/describe-media`. Common uses: describing an image for a downstream text model, describing an audio for accessibility, describing a video for a transcript.

## Objective

Produce a faithful textual description of the media, with the right level of detail and language coverage for the downstream task.

## Preconditions

- A description model is available on the account. Verify with `aivax_list_models`.
- The media is in a supported format.
- The account balance is sufficient for the modality (image/audio $0.10, file/video $0.50).

## Decision Tree

1. What is the modality? Image, audio, video, or file. Pick a model that supports it.
2. What is the level of detail? Short caption, paragraph description, or detailed enumeration. The level of detail affects the cost and the latency.
3. What is the language of the description? Match the user's language. The model may support multiple languages; verify with `aivax_list_models`.
4. Is the description for a downstream model or a human? For a downstream model, the description should be structured and dense. For a human, it should be readable and contextualized.
5. Should the description include specific aspects (e.g. text in image, faces, objects)? Some models support a `focus` parameter; verify with `aivax_search_context`.

## Construction

```text
{
  "model": "<description-model>",
  "file": "<url or data:>",
  "modality": "image",
  "detail": "standard",
  "language": "en",
  "metadata": { "trace_id": "tr_..." }
}
```

Some models accept `prompt` (a hint about what to describe), `max_length`, and `focus`. Verify with `aivax_search_context`.

## Response Handling

1. The response is a textual description.
2. Use the description for the downstream task: feed it to a model, store it, or display it.
3. Record the description length, the model, and the modality in the trace.

## Validation

- The description is faithful to the media. Spot-check on a sample.
- The level of detail matches the request.
- The language matches the request.
- The trace ID is preserved.

## Failure Modes

- The model returns no description: the media is unsupported, the balance is too low, or the content is unreachable. Inspect the error.
- The description is wrong: the model is not strong on the modality or the input is too noisy. Switch the model.
- The description is too short or too long: the parameters are wrong. Adjust.

## Escalation

- The description is for a downstream model: load `references/multimodal/situations/multimodal-preprocess.md` and consider preprocessing instead.
- The description is for accessibility: load `references/web-chat/` or another user-facing channel and integrate the description.
- The task is to describe many media files: load `references/batch/`.

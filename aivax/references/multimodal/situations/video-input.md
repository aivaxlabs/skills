# Video Input

Use when the user wants the model to reason about a video, either by sending the video directly to a multimodal model or by sampling key frames and sending them as images.

## Objective

Get the model's reasoning over video content with the right fidelity/cost trade-off, and the right choice between direct video input and frame sampling.

## Preconditions

- The model supports video input. Verify with `aivax_list_models`.
- The account balance is at least $0.50 (the minimum for video input).
- The video is reachable (URL) or can be base64-encoded (data URL).

## Decision Tree

1. Is the video shorter than ~1 minute? Send it directly via `video_url`. Lower latency, higher fidelity, but larger payload.
2. Is the video longer? Sample key frames (one per scene change or one per N seconds) and send them as `image_url` parts. The model reasons over the frames; the temporal dimension is approximated.
3. Is the video a screen recording, a lecture, or a meeting? Frame sampling with timestamps in the prompt is usually enough. The model can be told "frames from a 30-minute video, in order".
4. Is the video primarily audio (a podcast, a talk)? Transcribe the audio first and combine the transcript with a few key frames.
5. Is the video a security camera or similar? Sample frames at fixed intervals and reason over the sample. Sending the full video is rarely the right call.

## Construction

Direct:

```text
{
  "role": "user",
  "content": [
    { "type": "text", "text": "<prompt>" },
    { "type": "video_url", "video_url": { "url": "<url or data:>" } }
  ]
}
```

Frame sampling:

```text
1. Extract N key frames (e.g. one every 10 seconds) from the video
2. Build a user message with one image_url part per frame
3. Add a text part that names the timestamps
```

## Validation

- The model references the video (or frames) in its response.
- The `usage` block accounts for the video (or image) tokens.
- The trace ID is preserved.
- The model picked up the temporal cues (e.g. "the speaker raised their hand at 0:42").

## Failure Modes

- The model returns a content error: the video is not reachable, the format is unsupported, or the balance is below $0.50. Inspect the error and the resource.
- The model returns a generic answer: the prompt is too vague. Name the time range, the action of interest, or the speaker.
- The model returns a refusal: moderation is blocking. Do not bypass.

## Escalation

- The video is too large to send: sample frames and load `references/text-tools/situations/describe-media.md` to describe each frame.
- The task is repetitive across many videos: load `references/batch/situations/design-batch-workflow.md`.
- The video is a real-time stream: load `references/voice-realtime/` (for audio) or a streaming-aware path.

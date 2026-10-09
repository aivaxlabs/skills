# Text To Speech

Use when the agent must synthesize audio from text through `POST /api/v1/generations/speech`.

## Objective

Produce an audio file that matches the input text, in the requested voice, language, and format, with the right fidelity and cost.

## Preconditions

- A TTS model is available on the account. Discover it with `GET /api/v1/information/speech-models.json` or the media MCP's `list_models({ "type": "audio" })`.
- The account balance is sufficient.
- The agent has a place to store the audio (S3, GCS, local disk, the user's CMS).

## Decision Tree

1. What is the language? Pick a model that covers it natively. Multilingual models are a fallback when a native model is not available.
2. What is the voice? Pick a supported voice that matches the user's preference, or omit `voice` for the model default. Check the model documentation; the speech catalog does not list voice options.
3. What is the delivery path? REST supports `mp3`, `wav`, and `ogg` (default `wav`) as inline binary or base64 JSON. The media MCP returns a public MP3 URL; use it only when public hosting is appropriate.
4. Are tone or pacing instructions needed? Only MCP `generate_speech` exposes `instructions`; REST has no `speed` or `instructions` control. Verify the selected model supports the intended guidance.
5. Is the text long? Check [Limits](https://docs.aivax.net/docs/limits) and [Pricing](https://docs.aivax.net/docs/pricing). Split and concatenate if needed; do not assume this endpoint streams audio.
6. Does the user want SSML or voice customization? Check the selected model's documented capabilities before promising them.

## Construction

```text
{
  "model": "<tts-model>",
  "input": "<text>",
  "voice": "<voice-id>",
  "format": "mp3",
  "raw": true
}
```

Do not add `language`, `sample_rate`, `speed`, or `metadata` to this REST request; the handler does not consume them. For MCP, call `generate_speech` with `input`, `model`, optional `voice`, and optional `instructions` instead; do not send `raw` or `format`.

## Response Handling

1. With REST `raw: true`, read binary audio. Otherwise decode base64 from `data.data` and check `data.format` and `data.mimeType`. MCP instead returns text containing a public MP3 URL.
2. Store the audio only when the user's workflow requires it; verify the saved file and its content type.
3. Deliver the file or returned URL without promising an undocumented URL lifetime.
4. Record model, voice, and output location in the caller's trace, keeping sensitive input out of logs.

## Validation

- The audio is valid and plays in a standard player.
- The audio matches the input text. Spot-check the first few seconds.
- The voice and language match the request.
- The trace ID is preserved.

## Failure Modes

- The model returns no audio: the input was rejected (moderation, length). Adjust the input.
- The voice sounds wrong: the model does not support the requested voice or language. Switch the model.
- The audio is truncated: the input was too long. Split and concatenate.

## Escalation

- The user wants realtime two-way voice: load `references/voice-realtime/`.
- The user wants many audio outputs: load `references/batch/`.
- The user wants a custom voice: check the model's voice-cloning options. Custom voices usually require a pre-registration step.

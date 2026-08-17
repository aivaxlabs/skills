# Text To Speech

Use when the agent must synthesize audio from text through `POST /api/v1/generations/speech`.

## Objective

Produce an audio file that matches the input text, in the requested voice, language, and format, with the right fidelity and cost.

## Preconditions

- A TTS model is available on the account. Verify with `aivax_list_models`.
- The account balance is sufficient.
- The agent has a place to store the audio (S3, GCS, local disk, the user's CMS).

## Decision Tree

1. What is the language? Pick a model that covers it natively. Multilingual models are a fallback when a native model is not available.
2. What is the voice? Pick a voice that matches the brand or the user's preference. The catalog usually lists the available voices per model.
3. What is the output format? MP3 is the most portable. WAV is higher fidelity. OGG and Opus are smaller.
4. What is the speed? 1.0 is normal. Faster is better for notifications; slower is better for clarity.
5. Is the text long? Long inputs may hit a per-request character limit. Split into segments and concatenate, or use a streaming TTS endpoint.
6. Does the user want SSML or prosody control? Verify the model supports it with `aivax_search_context` before relying on it.

## Construction

```text
{
  "model": "<tts-model>",
  "input": "<text>",
  "voice": "<voice-id>",
  "format": "mp3",
  "speed": 1.0,
  "metadata": { "trace_id": "tr_..." }
}
```

Some models accept `language` (when not auto-detected) and `sample_rate`. Verify with `aivax_search_context`.

## Response Handling

1. The response is an audio file in the requested format.
2. Store the audio in the user's storage of choice.
3. Pass the stable URL to the user or the downstream system.
4. Record the model, voice, input, and audio URL in the trace.

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

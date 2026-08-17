# Transcribe Audio

Use when the agent must transcribe an audio file to text through `POST /api/v1/generations/transcribe-audio`.

## Objective

Produce a transcript that is faithful to the audio, with the right language, speaker labels (when supported), and output format for downstream use.

## Preconditions

- A STT model is available on the account. Verify with `aivax_list_models`.
- The audio is in a supported format (e.g. `wav`, `mp3`, `m4a`, `flac`).
- The account balance is sufficient.
- The agent has a place to store the transcript.

## Decision Tree

1. What is the language? Set `language` explicitly when known. Auto-detect only when the language is unknown.
2. Is the audio multi-speaker? Enable diarization when supported. The output will include speaker labels.
3. Is the audio long (>10 minutes)? Split into segments. Some models have a per-request duration limit.
4. Does the user want timestamps? Set `response_format: "verbose_json"` or the equivalent when supported. The output will include start and end times per segment.
5. Does the user want a custom vocabulary? Some models accept a `prompt` with key terms to bias recognition. Verify with `aivax_search_context`.

## Construction

```text
{
  "model": "<stt-model>",
  "file": "<url or data:>",
  "language": "<iso-code>",
  "response_format": "verbose_json",
  "metadata": { "trace_id": "tr_..." }
}
```

Some models accept `diarize: true` for speaker labels and `prompt` for custom vocabulary.

## Response Handling

1. The response is a transcript in the requested format.
2. Normalize the transcript for downstream use: strip filler words, fix obvious punctuation, identify speakers.
3. Store the normalized transcript in the user's storage of choice.
4. Record the model, language, parameters, and the transcript location in the trace.

## Validation

- The transcript is faithful to the audio. Sample a few segments.
- Speaker labels (when enabled) are accurate.
- Timestamps (when enabled) align with the audio.
- The trace ID is preserved.

## Failure Modes

- The model returns no transcript: the audio is unsupported, the language is wrong, or the audio is too long. Inspect the error.
- The transcript is wrong: the model is not strong on the language or accent. Switch the model or set a custom vocabulary.
- Speaker labels are wrong: the model does not support diarization, or the audio has overlapping speakers. Try a different model.

## Escalation

- The user wants realtime transcription: load `references/voice-realtime/`.
- The user wants to transcribe many audio files: load `references/batch/`.
- The user wants the model to reason about the audio, not just transcribe it: load `references/multimodal/situations/audio-input.md`.

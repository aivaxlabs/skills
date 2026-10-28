# Transcribe Audio

Use when the agent must transcribe an audio file to text through `POST /api/v1/generations/audio/transcriptions`.

## Objective

Produce a transcript that is faithful to the audio, with the right language, speaker labels (when supported), and output format for downstream use.

## Preconditions

- A STT model is available in `GET /api/v1/information/audio-transcription-models.json` (`aivax_list_models` is the text catalog, not this one). `model` is optional; omitting it uses the catalog's default.
- The audio is in a format listed in the catalog's `supportedFormats` and is sent as base64 (`wav`, `mp3`, `m4a`, `flac`, `ogg`, `webm`, or `aac`). URLs and `data:` strings are not accepted.
- The account balance is sufficient.
- The agent has a place to store the transcript.

## Decision Tree

1. What is the language? Set `language` explicitly when known. Auto-detect only when the language is unknown.
2. Is the audio multi-speaker, or are timestamps or a custom vocabulary needed? This endpoint exposes none of them (no diarization, `response_format`, or `prompt`); the response is plain `text`, even for models whose catalog description mentions timestamps or speaker diarization. If those are required, say so instead of sending unsupported fields.
3. Is the audio long? Check the request size and duration limits in [Limits](https://docs.aivax.net/docs/limits) and split into segments when needed. Billing is by audio duration.

## Construction

```text
{
  "model": "<stt-model>",
  "input_audio": { "data": "<base64>", "format": "wav" },
  "language": "<iso-code>"
}
```

`model` and `language` are optional. Do not add `file`, `response_format`, `metadata`, `diarize`, or `prompt`; the handler does not read them.

## Response Handling

1. The standard envelope's `data` carries `text` and `usage` (`cost`, `process_time`).
2. Normalize the transcript for downstream use: strip filler words and fix obvious punctuation, keeping the original text when fidelity matters.
3. Store the normalized transcript in the user's storage of choice.
4. Record the model, language, parameters, and the transcript location in the trace.

## Validation

- The transcript is faithful to the audio. Sample a few segments.
- The trace ID is preserved.

## Failure Modes

- 400: unknown `model`, unsupported `input_audio.format`, empty or invalid base64, or audio that cannot be decoded or has no measurable duration. Fix the payload; do not retry unchanged.
- The transcript is wrong: the model is not strong on the language or accent. Set `language` or switch the model.

## Escalation

- The user wants realtime transcription: load `references/voice-realtime/`.
- The user wants to transcribe many audio files: load `references/batch/`.
- The user wants the model to reason about the audio, not just transcribe it: load `references/multimodal/situations/audio-input.md`.

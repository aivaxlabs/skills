# Audio Input

Use when the user wants the model to reason about audio content, either by sending the audio directly to a multimodal model or by transcribing it first and sending the transcript as text.

## Objective

Get the model's reasoning over audio content with the right fidelity/cost trade-off, and the right choice between direct audio input and transcription.

## Preconditions

- The audio is in a supported format (e.g. `wav`, `mp3`).
- The account balance meets the minimum for audio input ($0.10) or the cost of the chosen alternative path.
- The model supports the chosen path.

## Decision Tree

1. Does the model support `input_audio`? If yes, send the audio directly. Lower cost for short audio; higher fidelity for tone, timing, and non-speech signals.
2. Does the model not support audio, or is the audio primarily speech? Transcribe first with `POST /api/v1/generations/transcribe-audio` and pass the transcript as text. The model can reason over the transcript without seeing the audio.
3. Is the audio long (>10 minutes)? Transcription is usually cheaper. Compare the per-minute transcription cost against the per-token audio cost of the multimodal model.
4. Is the audio non-speech (music, environmental sound, sound effects)? Direct audio is usually the only path. Text transcription of these is lossy.
5. Is the audio a recording that must be re-summarized? Direct audio is usually better; the model can pick up tone and emphasis.

## Construction

Direct:

```text
{
  "role": "user",
  "content": [
    { "type": "text", "text": "<prompt>" },
    { "type": "input_audio", "input_audio": { "data": "<base64>", "format": "wav" } }
  ]
}
```

Transcribe-then-reason:

```text
1. POST /api/v1/generations/transcribe-audio with the audio
2. Receive transcript
3. POST /v1/chat/completions with the transcript in a user message
```

## Validation

- The model references the audio in its response.
- The `usage` block includes audio tokens (direct) or transcription tokens (transcribe-then-reason).
- The trace ID is preserved.
- The transcript (when used) is faithful to the audio. Sample a few segments.

## Failure Modes

- Unsupported format: convert to a supported format before sending.
- Audio too long: split into segments. Each segment gets a separate transcription or audio input.
- Transcription mistranscribes names or jargon: post-process the transcript with a small text model before sending to the main model.

## Escalation

- The audio is a multi-speaker conversation: load `references/speech/situations/transcribe-audio.md` to see the transcription options for speaker labels.
- The audio is a real-time stream: load `references/voice-realtime/` instead of this sub-skill.

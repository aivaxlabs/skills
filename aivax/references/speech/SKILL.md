---
name: aivax-speech
description: Synthesize speech from text (TTS) and transcribe audio to text (STT) with AIVAX. Load when the user wants a text-to-speech output, when audio must be transcribed for downstream reasoning, or when the agent is building a voice-capable workflow outside realtime channels.
---

# Speech

This sub-skill owns text-to-speech and speech-to-text generation through AIVAX. Realtime bidirectional voice lives in `references/voice-realtime/`; this sub-skill is for one-way or batch use.

## Operating Files

- `situations/text-to-speech.md`: synthesize audio from text.
- `situations/transcribe-audio.md`: transcribe audio to text.

## Endpoints

- Text-to-speech: `POST /api/v1/generations/speech`.
- Speech-to-text: `POST /api/v1/generations/transcribe-audio`.

Both endpoints accept AIVAX API keys (private or public, depending on the workflow). The model and parameters are part of the request body. Verify the model and parameter shape with `aivax_search_context` or the public docs before relying on a field.

## When To Use Speech

Use this sub-skill when the user wants:

- A text reply rendered as audio (e.g. an audio message, a podcast segment, a notification tone).
- An audio file transcribed to text for downstream reasoning, indexing, or routing.
- A batch transcription or generation.

Do not use this sub-skill for:

- Realtime bidirectional voice (load `references/voice-realtime/`).
- Audio input inside a chat completion (load `references/multimodal/situations/audio-input.md`).
- Audio inside a chat client or web chat (load `references/web-chat/`).

## Models

Speech models vary by language coverage, voice quality, latency, and cost. Verify with `aivax_list_models`. Common axes:

- Language coverage: multilingual models cover more languages but may be weaker per language than a monolingual model.
- Voice quality: some models are tuned for natural-sounding speech, others for clarity or speed.
- Latency: lower-latency models are better for interactive use; higher-latency models are usually better for batch.
- Cost: TTS is priced per character or per audio second; STT is priced per audio second.

## Cost Awareness

Both endpoints have per-account quotas. Verify before running large jobs. For batch use, load `references/batch/`.

## Validation

- TTS: the response is a valid audio file in the requested format. The audio matches the input text.
- STT: the response is a transcript that is faithful to the audio. Sample a few segments.
- The trace ID is preserved through `metadata`.

## Limits

- The account balance must meet the minimum for media generation (typically $0.10 for audio).
- Some models support only a subset of languages or voices. Verify with `aivax_list_models`.
- Long audio can hit a per-request duration limit. Split into segments when needed.

## Escalation

- The TTS output sounds wrong: try a different model or voice. Load `situations/text-to-speech.md` for the voice parameter.
- The STT output is wrong: load `situations/transcribe-audio.md` for language and diarization parameters.
- The task needs realtime two-way voice: load `references/voice-realtime/`.
- The task is batch transcription: load `references/batch/`.

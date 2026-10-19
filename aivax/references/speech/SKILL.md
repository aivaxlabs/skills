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
- Speech-to-text: `POST /api/v1/generations/audio/transcriptions`.

Both endpoints accept AIVAX API keys (private or public, depending on the workflow).

- TTS accepts `model`, `input`, optional `voice`, optional `format` (`mp3`, `wav`, or `ogg`; default `wav`), and optional `raw`. With the default `raw=false`, it returns the standard JSON envelope whose `data` contains `{ "format", "mimeType", "data" }`, with audio base64 in `data`. Binary audio is returned only with `raw=true`. Do not send MCP-only `instructions`, or invent `speed`, `language`, `sample_rate`, or `metadata` controls for this REST endpoint.
- STT accepts only optional `model`, required `input_audio.data`, required `input_audio.format`, and optional `language`; diarization is not a request parameter (`OpenIndexer.Alpha2/Api/Modules/Generations/AudioTranscriptionsApi.cs:58-76`). Use `GET /api/v1/information/audio-transcription-models.json` for the live STT catalog (`OpenIndexer.Alpha2/Api/Modules/InformationApi.cs:63`).

## Media Generation MCP

When configured, use the Streamable HTTP server at `https://inference.aivax.net/v1/mcp/media-generation`, authenticated with an account bearer key. Keep credentials in the client's secret storage.

- Discover TTS models with `list_models({ "type": "audio" })`, then call `generate_speech` with required `input` and `model`, and optional `voice` and `instructions` (tone, emotion, pacing).
- The result is MCP text containing a public MP3 URL, not inline audio or the REST JSON envelope. It has no `format` or `raw` option. Prefer REST when public hosting is unsuitable or WAV/OGG output is needed.
- Narrow discovery with `X-Mcp-Enabled-Tools: list_models, generate_speech`. Omitting the header exposes all media tools; an empty or unrecognized allowlist exposes none. Refresh MCP discovery after changing it.
- This server does not transcribe audio; use the STT endpoint for that task. Generation remains billable; consult [Pricing](https://docs.aivax.net/docs/pricing) and [Limits](https://docs.aivax.net/docs/limits).

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

Speech models vary by language coverage, voice quality, latency, and cost. Verify TTS models with `GET /api/v1/information/speech-models.json` or the media MCP's `list_models` with `type: "audio"`; use `GET /api/v1/information/audio-transcription-models.json` for STT. Do not confuse these with the text-model catalog. TTS `model` is the catalog `name` (an enum-style identifier, not a provider slug); STT `model` is the catalog `name` and defaults to the catalog entry flagged `isDefault` when omitted. Both catalogs change, so re-read them rather than hardcoding names. Some TTS models take inline speech direction (for example bracketed cues before a line); the catalog `description` says which, and only those models should receive such cues. Common axes:

- Language coverage: multilingual models cover more languages but may be weaker per language than a monolingual model.
- Voice quality: some models are tuned for natural-sounding speech, others for clarity or speed.
- Latency: lower-latency models are better for interactive use; higher-latency models are usually better for batch.
- Cost: TTS is priced per character or per audio second; STT is priced per audio second.

## Cost Awareness

Both endpoints have per-account quotas. Verify before running large jobs. For batch use, load `references/batch/`.

## Validation

- TTS with `raw=false`: decode the base64 `data.data` and verify that its format matches `data.format` and `data.mimeType`.
- TTS with `raw=true`: verify the binary response and its content type.
- STT: the response is a transcript that is faithful to the audio. Sample a few segments.

## Limits

- Check current balance requirements and quotas in [Limits](https://docs.aivax.net/docs/limits), and charges in [Pricing](https://docs.aivax.net/docs/pricing).
- Some models support only a subset of languages or voices. Verify the selected speech model's documentation.
- Long audio can hit a per-request duration limit. Split into segments when needed.

## Escalation

- The TTS output sounds wrong: try a different model or voice. Load `situations/text-to-speech.md` for the voice parameter.
- The STT output is wrong: load `situations/transcribe-audio.md` for the optional language hint.
- The task needs realtime two-way voice: load `references/voice-realtime/`.
- The task is batch transcription: load `references/batch/`.
- The request fails with 429 or a transient error: load `references/resilience/situations/rate-limit-429.md`.

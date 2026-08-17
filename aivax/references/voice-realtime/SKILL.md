---
name: aivax-voice-realtime
description: Open and operate realtime bidirectional voice sessions on AIVAX. Load when the user wants a streaming, two-way voice interaction with low latency — for example, an in-app voice assistant, a phone-style conversation flow, or a voice-driven tool use scenario.
---

# Voice Realtime

This sub-skill owns realtime bidirectional voice sessions on AIVAX. The endpoint is `GET /v1/realtime-voice/sessions`. The discipline is opening a session with the right model, voice, and instructions, and then managing the lifecycle (connect, send audio, receive audio, handle interruptions, close).

## Operating Files

- `situations/open-realtime-session.md`: open a realtime voice session and pick the model and voice.
- `situations/design-realtime-flow.md`: design the conversation flow, tool use, and interruption handling.

## When To Use Voice Realtime

Use this sub-skill when the user wants:

- A two-way voice conversation with a model.
- Low-latency responses suitable for interactive use.
- Tool use inside a voice conversation (e.g. "look up my order").
- Realtime interruptions (the user can cut the model off mid-sentence).
- Audio transcription or synthesis as a side effect of a voice conversation.

Do not use this sub-skill for:

- One-way TTS or STT (load `references/speech/`).
- Audio inside a chat completion (load `references/multimodal/situations/audio-input.md`).
- Web chat with audio upload (load `references/web-chat/`).

## Endpoint

`GET /v1/realtime-voice/sessions` opens a session. The response includes the session ID, the model, the voice, and the connection details (transport, sample rate, encoding). The exact fields depend on the model and the AIVAX version; verify with `aivax_search_context` before relying on a field.

## Models

Realtime models are tuned for low-latency two-way audio. They differ in language coverage, voice quality, tool use support, and cost. Verify with `aivax_list_models` and `aivax_search_context`. Common axes:

- Voice quality vs. latency: higher quality usually means higher latency.
- Multilingual support: pick a model that covers the user's language.
- Tool use inside the voice session: not all realtime models support function calling.
- Interruption handling: most do, but the API for sending the interruption event varies.

## Session Lifecycle

1. Open the session: `GET /v1/realtime-voice/sessions` with the model, voice, instructions, and any tools.
2. Connect: the response includes the transport (WebSocket, WebRTC, or another). Connect with the credentials from the response.
3. Send audio: stream the user's microphone audio to the session.
4. Receive audio: stream the model's response audio to the user.
5. Handle events: tool calls, interruptions, errors, and end-of-turn signals.
6. Close: send the close event or close the transport.

## Cost Awareness

Realtime voice is priced per second of audio. A long conversation can cost more than a chat equivalent. Set a cost cap with the user before opening a long session. Some realtime models charge for both input and output audio; verify with `aivax_list_models`.

## Validation

- The session opens with the chosen model and voice.
- The audio round-trip is below the user's latency budget.
- Tool calls inside the session are dispatched and re-fed correctly.
- The trace ID is preserved.
- The session closes cleanly and the cost is within the user's cap.

## Limits

- Realtime voice has a per-account quota. Long sessions can hit the quota.
- Some models support only a subset of voices or languages.
- The session token is short-lived. Do not store it; reconnect when it expires.

## Escalation

- The latency is too high: switch to a faster model, reduce the audio quality, or use a lower-latency transport.
- The cost is too high: lower the audio quality, shorten the session, or switch to a cheaper model.
- The user wants a one-way TTS or STT: load `references/speech/`.
- The user wants to combine voice with web chat: load `references/web-chat/`.

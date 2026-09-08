# Open Realtime Session

Use when the agent must open a realtime voice session through `GET /api/v1/voice-session`.

## Objective

Open a session with the right model, voice, and instructions, and connect the transport with the right credentials.

## Preconditions

- A realtime model is available on the account. Verify with `aivax_list_models`.
- The account balance is sufficient.
- The user has consented to the session and the audio capture.

## Decision Tree

1. What is the language? Pick a model that covers it natively.
2. What is the voice? Pick a voice that matches the brand or the user's preference.
3. What is the latency budget? Realtime models differ in latency. Pick a faster one when the budget is tight.
4. Does the session need tool use? Pick a model that supports function calling in realtime.
5. Does the session need interruptions? Most realtime models support them; verify the API for sending the interruption event.
6. The endpoint uses WebSocket with the OpenAI-compatible Realtime event protocol, not WebRTC. Use a server-side connection or relay so the private API key is not exposed in a browser.

## Construction

```text
GET /api/v1/voice-session
Authorization: Bearer <private-api-key>
Connection: Upgrade
Upgrade: websocket
```

Use a WebSocket client to complete the upgrade handshake. The first client message must be:

```json
{
  "type": "session.update",
  "gateway": "<gateway-selector>",
  "model": "<realtime-model>",
  "session": {
    "audio": { "output": { "voice": "<voice>" } },
    "instructions": "<session-instructions>",
    "tools": []
  }
}
```

`gateway` and `model` are optional; omit them to use server defaults. They may also appear inside `session`, but event-level values take precedence. The server removes these selectors before forwarding the event. Standard Realtime settings, including `reasoning.effort`, belong inside `session`; they are not query-string parameters.

## Response Handling

1. A successful upgrade returns `101 Switching Protocols`, not a JSON connection descriptor.
2. Send the initial `session.update` event containing a `session` object. Public API keys are rejected.
3. Process OpenAI-compatible Realtime events and stream audio on that same WebSocket using the configured format.
4. Preserve request and session identifiers when available for tracing, without logging credentials. Close the WebSocket when done.

## Validation

- The session opens with the chosen model and voice.
- The transport connects without error.
- The audio round-trip is below the user's latency budget.
- The trace ID is preserved.

## Failure Modes

- The model is not available on the plan: switch the model.
- The transport fails to connect: the network or the credentials are wrong. Inspect the error and the network.
- The initial event is rejected: ensure it is valid JSON with `type: session.update` and a `session` object. Each new WebSocket connection requires this event.

## Escalation

- The latency is too high: load `situations/design-realtime-flow.md` and reduce the audio quality or switch to a faster model.
- The cost is too high: shorten the session or switch to a cheaper model.
- The session needs to be combined with web chat: load `references/web-chat/`.

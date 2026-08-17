# Open Realtime Session

Use when the agent must open a realtime voice session through `GET /v1/realtime-voice/sessions`.

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
6. What is the transport? WebSocket is the most common. WebRTC is better for browser-based use cases. The model may dictate the transport; verify with `aivax_search_context`.

## Construction

```text
GET /v1/realtime-voice/sessions?model=<model>&voice=<voice>&language=<lang>
Authorization: Bearer <api-key>
```

Some models accept `instructions`, `tools`, and other parameters in the query string or the body. Verify with `aivax_search_context`.

## Response Handling

1. The response includes the session ID, the model, the voice, the transport type, and the connection details.
2. Connect to the transport with the credentials from the response. Do not log the credentials.
3. The session is now open. Begin streaming audio.
4. Record the session ID and the model in the trace.

## Validation

- The session opens with the chosen model and voice.
- The transport connects without error.
- The audio round-trip is below the user's latency budget.
- The trace ID is preserved.

## Failure Modes

- The model is not available on the plan: switch the model.
- The transport fails to connect: the network or the credentials are wrong. Inspect the error and the network.
- The session token is short-lived: reconnect when it expires.

## Escalation

- The latency is too high: load `situations/design-realtime-flow.md` and reduce the audio quality or switch to a faster model.
- The cost is too high: shorten the session or switch to a cheaper model.
- The session needs to be combined with web chat: load `references/web-chat/`.

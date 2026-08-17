# Design Realtime Flow

Use when the agent must design the conversation flow of a realtime voice session: turn-taking, tool use, interruptions, error handling, and session close.

## Objective

Produce a session that feels natural to the user, that handles interruptions and tool use correctly, and that closes cleanly when the user is done.

## Preconditions

- The session is open (load `situations/open-realtime-session.md`).
- The model and voice are chosen.
- The instructions for the model are drafted.

## Decision Tree

1. What is the turn-taking model? Push-to-talk is the simplest. Open-mic with voice-activity-detection is more natural but harder to implement well.
2. Does the session need tool use? Pass the tool definitions when opening the session. The model will call them as needed.
3. Does the session need interruptions? Most realtime models support them. Implement the interruption event and the model will stop speaking.
4. How long should the session last? Set a max duration and a max cost. Close the session when either is reached.
5. How should the session end? On a user-initiated close, on a model-initiated close, or on a timeout.
6. What is the post-session flow? Transcribe the session, store the transcript, charge the user, log the conversation.

## Turn-Taking

- **Push-to-talk**: the user holds a button to speak. The simplest model, the worst UX.
- **Voice activity detection**: the model detects when the user is speaking and when they stop. Better UX, more complex.
- **Interruption handling**: the user can cut the model off mid-sentence. The model stops and listens. Most realtime models support this out of the box.

## Tool Use Inside Voice

- Define the tools when opening the session. The model will call them as needed.
- The tool result is fed back to the model as text or audio. Text is faster and cheaper.
- Tool calls that have external side effects (sending email, publishing a page, deleting a resource) require explicit user approval. Confirm before dispatching.

## Error Handling

- Network errors: the session may be temporarily disconnected. Implement a reconnect with exponential backoff.
- Audio errors: the user's microphone may be muted, the model's audio may be too loud, or the audio codec may be unsupported. Inspect the error and adjust.
- Model errors: the model may return a refusal, a content filter, or an unknown error. Surface the error to the user and offer to retry.

## Session Close

- The user says "goodbye" or pushes a close button.
- The model signals end-of-turn and the session closes.
- A timeout is reached.
- A cost cap is reached.

The agent must close the session cleanly, store the transcript (if any), charge the user, and log the conversation.

## Cost Awareness

Realtime voice is priced per second. A 30-minute conversation can cost more than a chat equivalent. Set a max duration and a max cost with the user before opening a long session.

## Validation

- The session opens and closes cleanly.
- The audio round-trip is below the user's latency budget.
- Tool calls inside the session are dispatched and re-fed correctly.
- Interruptions are handled gracefully.
- The session ends within the cost and duration caps.
- The transcript (if any) is stored.

## Escalation

- The session is consistently slow: load `references/observability/situations/diagnose-degradation.md`.
- The session is consistently expensive: load `references/cost-monitoring/situations/investigate-cost-spike.md`.
- The session needs to be combined with web chat: load `references/web-chat/`.
- The session needs to be persisted: load `references/observability/situations/audit-conversation.md`.

---
name: aivax-web-chat
description: Operate AIVAX web chat clients, browser sessions, embed settings, limits, and messaging integrations. Load when the agent must create or edit a user-facing chat client, issue or inspect a session, configure embedding and media inputs, or diagnose a client-specific issue.
---

# Web Chat

This sub-skill owns the user-facing surfaces that connect a browser or messaging channel to an AIVAX AI Gateway. A chat client controls UI behavior, allowed frame origins, rate limits, sessions, media upload modes, audio synthesis, and optional messaging integrations.

## Operating Files

- `situations/create-chat-client.md`: create a client linked to a gateway with safe browser-facing defaults.
- `situations/manage-session.md`: create, refresh, and inspect a session without exposing access credentials.
- `situations/edit-client-safely.md`: change one client parameter, limit, or integration setting without replacing unrelated configuration.

## When To Use Web Chat

Use this sub-skill to:

- Create, inspect, or edit a web chat client.
- Configure the initial user experience, allowed iframe origins, input modes, audio synthesis, or rate limits.
- Create or refresh a customer-specific session and retrieve its `talkUrl`.
- Configure or remove a Zapi, Telegram, EvolutionApi, or Kapso integration.
- Diagnose a problem scoped to one chat client or session.

Do not use this sub-skill to tune the model, tools, RAG, or instructions behind the chat. Use `references/ai-gateways/`, `references/rag/`, or `references/text-inference/` for those causes.

## Operating Surfaces

- `GET /api/v1/web-chat-client`: list clients.
- `POST /api/v1/web-chat-client`: create a client linked to a gateway.
- `GET /api/v1/web-chat-client/<id>`: inspect a client, including limits, parameters, and integrations.
- `PUT /api/v1/web-chat-client/<id>`: update a client.
- `GET /api/v1/web-chat-client/<id>/sessions`: list sessions.
- `POST /api/v1/web-chat-client/<id>/sessions`: create or refresh a session.
- `PUT /api/v1/web-chat-client/<id>/integrations`: configure a messaging integration.
- `DELETE /api/v1/web-chat-client/<id>/integrations/<type>`: remove one integration (destructive).

Read the client before changing it. For unfamiliar integration payloads, scheduled-continuation fields, or audio-synthesis options, use `aivax_search_context` before mutating.

## Client Parameters

The main client parameters include:

- `primaryColor`, `pageTitle`, `logoImageUrl`, `helloLabel`, `helloSubLabel`, and `textAreaPlaceholder` for presentation.
- `suggestionButtons` for high-value starting prompts.
- `allowedFrameOrigins` for embedding control.
- `inputModes` (`Image`, `Document`, `Audio`) and `uploadUnsupportedFiles` for uploads.
- `showToolCalls` and `debug` for diagnostics; disable `debug` after troubleshooting.
- `audioSynthesisSource`, `audioSynthesisVoice`, `audioSynthesisInstruction`, and `summarizeTextBeforeAudioSynthesis` for spoken replies.
- `allowScheduledContinuations`, `splitAnswerIntoMessageChunks`, `maxScheduledIgnoredZone`, and `messageDebounceInterval` for messaging-channel behavior.

The principal limits are `messagesPerHour` and `maxMessages`. Set them deliberately: overly permissive values increase cost and abuse exposure, while overly restrictive values harm the user experience.

## Session Hygiene

A session response can contain `sessionId`, `accessKey`, and `talkUrl`. Treat all three as sensitive operational data. Do not include them in a final response unless the user needs the exact value to continue the requested task. Do not store secrets or sensitive personal data in `extraContext` or session metadata.

## Validation

- The client references the intended gateway.
- Limits and changed parameters match the intended values; unrelated configuration is preserved.
- The permitted embedding origin works, and an unpermitted origin is not added accidentally.
- Session creation or refresh returns a usable result without exposing access material.
- Recent conversations show the expected client or session attribution.

## Escalation

- The issue appears in the underlying response: load `references/observability/` to trace it, then the implicated capability skill.
- The gateway behavior is wrong: load `references/ai-gateways/`.
- The session has rate-limit or transient failures: load `references/resilience/`.
- The client is causing unexpected spend: load `references/cost-monitoring/situations/investigate-cost-spike.md`.

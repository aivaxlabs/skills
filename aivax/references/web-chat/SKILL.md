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
- Configure or remove a Zapi, Telegram, EvolutionApi, or Kapso integration, or rotate its webhook key.
- Send prompts to a session, with per-inference instructions, client-side tools, streaming, or inference options.
- Diagnose a problem scoped to one chat client or session.

Do not use this sub-skill to tune the model, tools, RAG, or instructions behind the chat. Use `references/ai-gateways/`, `references/rag/`, or `references/text-inference/` for those causes.

## Operating Surfaces

- `GET /api/v1/web-chat-client`: list clients.
- `POST /api/v1/web-chat-client`: create a client linked to a gateway.
- `GET /api/v1/web-chat-client/<id>`: inspect a client, including limits, parameters, and integrations.
- `PUT /api/v1/web-chat-client/<id>`: update a client.
- `DELETE /api/v1/web-chat-client/<id>`: delete a client (destructive; require explicit confirmation and verify it is absent afterward).
- `GET /api/v1/web-chat-client/<id>/sessions?filter=<text>`: list up to 300 sessions; `filter` searches access key, extra context, external user ID, or exact session ID.
- `POST /api/v1/web-chat-client/<id>/sessions`: create or refresh a session.
- `DELETE /api/v1/web-chat-client/<id>/sessions/<session-id>`: revoke access by ending and deleting the session (destructive; require explicit confirmation).
- `PUT /api/v1/web-chat-client/<id>/integrations`: configure a messaging integration.
- `DELETE /api/v1/web-chat-client/<id>/integrations/<name>`: remove one integration (destructive).
- `POST /api/v1/web-chat-client/<id>/integrations/<name>/hook-key`: rotate the webhook key of one configured integration (`Zapi`, `Telegram`, `EvolutionApi`, or `Kapso`) and return the new `hookKey`. The webhook URL with the previous key stops working immediately; 404 when the integration is not configured.
- `POST /api/v1/public/chat-clients/<access-key>/prompt`: send a prompt as the session holder. See Session Prompts.

Read the client before changing it. For unfamiliar integration payloads, scheduled-continuation fields, or audio-synthesis options, use `aivax_search_context` before mutating.

## Client Parameters

The main client parameters include:

- `primaryColor`, `pageTitle`, `logoImageUrl`, `helloLabel`, `helloSubLabel`, and `textAreaPlaceholder` for presentation.
- `suggestionButtons` for high-value starting prompts.
- `allowedFrameOrigins` for embedding control.
- `inputModes` (`Image`, `Document`, `Audio`) and `uploadUnsupportedFiles` for uploads.
- `showToolCalls` and `debug` for diagnostics; disable `debug` after troubleshooting.
- `audioSynthesisSource`, `audioSynthesisVoice`, `audioSynthesisInstruction`, and `summarizeTextBeforeAudioSynthesis` for spoken replies. `audioSynthesisSource` takes a TTS model `name` from `GET /api/v1/information/speech-models.json`, or `None` to disable audio.
- `allowScheduledContinuations`, `splitAnswerIntoMessageChunks`, `maxScheduledIgnoredZone`, and `messageDebounceInterval` for messaging-channel behavior.
- On messaging channels, a user message that arrives while a reply is being generated is steered into the running inference between its rounds, so the agent can react without waiting for a new turn. Messages not delivered before the run ends start the next generation after the debounce interval. `/reset` and `/usage` are not steered.

## Integration Webhooks

Each configured integration has a server-generated `hookKey`, returned in the client detail; treat it as a secret. Webhook URLs carry it as a path segment:

- Zapi: `/api/v1/intl/zapi/<client-id>/<hook-key>/on-message-received`
- Telegram: `/api/v1/intl/tg/<client-id>/<hook-key>/hook`
- EvolutionApi: `/api/v1/intl/evolution-api/<client-id>/<hook-key>/hook`
- Kapso: `/api/v1/intl/kapso/<client-id>/<hook-key>/hook`

A wrong key returns 401. The legacy URLs without `<hook-key>` keep working until 2026-12-01 and then return 410; move providers to the keyed URL before that date. Clients configured earlier receive a key on the next client read or integration update.

Provider signatures are also verified:

- Kapso: when `webhookSecretKey` is set, `X-Webhook-Signature` must be the lowercase hex HMAC-SHA256 of the raw body with that secret, or the request returns 401. Media downloads are accepted only from the Kapso API host.
- Telegram: `POST /api/v1/intl/tg/<client-id>/update` registers the keyed URL with Telegram and sets `secret_token` to the hook key. On the keyed route, `X-Telegram-Bot-Api-Secret-Token` must match, or the request returns 401. Call it again after every hook-key rotation.

After rotating a key, update the webhook URL in the provider (for Telegram, call `update`) before relying on the channel.

## Rate Limits

The principal limits are `messagesPerHour` and `maxMessages`. Set them deliberately: overly permissive values increase cost and abuse exposure, while overly restrictive values harm the user experience.

## Session Hygiene

A session response can contain `sessionId`, `accessKey`, and `talkUrl`. Treat all three as sensitive operational data. Do not include them in a final response unless the user needs the exact value to continue the requested task. Do not store secrets or sensitive personal data in `extraContext` or session metadata.

`expires` is measured in seconds, accepts 600 through 7,776,000 (90 days), and defaults to 3,600. A non-empty `tag` is idempotent per client while its session is not expired: creating with the same tag restores that session and replaces its context, expiration, and metadata instead of creating another one.

## Session Prompts

`POST /api/v1/public/chat-clients/<access-key>/prompt` runs one inference on the session history. Fields:

- `prompt` (required): text, one OpenAI-compatible message, or an array of messages, including `tool` results for client-side tool calls.
- `commit`: default `true`. `false` runs against the history without saving this turn; usage and tool side effects still occur.
- `instructions`: string or array of strings for this inference only. It is not persisted and does not replace the session `extraContext`.
- `tools`: OpenAI-compatible function tools for this inference only, each with `function.name`. They are client-side tools: calls return in `toolCalls` for your application to execute. They stay visible even when the gateway hides tools without a skill. Do not repeat gateway tool names, and send them again with the tool results.
- `stream`: default `false`. `true` returns server-sent events: `text` (`content`, `replace`), `reasoning`, `tool` (server-side tool updates), a final `result` with the same body as the JSON response, or `error` with a `message`.
- `inferenceOptions`: `reasoningEffort`, `toolInvocationExplanations`, and `renderingMode` (`default` or `textual_blocks`; other values are rejected). Omitted fields keep the gateway configuration.

The response has `completionText`, `reasoning`, client-side `toolCalls`, `usage`, and `createdMessages`.

The session `extraContext`, the content fetched from `contextLocation`, and `instructions` are joined into the system instructions of each inference, in that order. `contextLocation` is fetched before every prompt; a failed fetch is logged as an account warning and the inference continues without it. None of them is saved to the session history.

## Validation

- The client references the intended gateway.
- Limits and changed parameters match the intended values; unrelated configuration is preserved.
- The permitted embedding origin works, and an unpermitted origin is not added accidentally.
- Session creation or refresh returns a usable result without exposing access material.
- After client deletion, `GET /api/v1/web-chat-client/<id>` no longer returns the client; after session revocation, the session no longer appears in its client's listing.
- Recent conversations show the expected client or session attribution.

## Escalation

- The issue appears in the underlying response: load `references/observability/` to trace it, then the implicated capability skill.
- The gateway behavior is wrong: load `references/ai-gateways/`.
- The session has rate-limit or transient failures: load `references/resilience/`.
- The client is causing unexpected spend: load `references/cost-monitoring/situations/investigate-cost-spike.md`.

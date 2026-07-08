# Chat Clients

Use this sub-skill to manage the user-facing surfaces that connect people or messaging channels to an AIVAX AI Gateway. A chat client controls UI behavior, allowed origins, rate limits, sessions, media upload modes, and optional WhatsApp or Telegram integrations.

This is for configuring chat clients and sessions in the user's AIVAX account. Do not change AIVAX chat UI/source code or integration implementation code unless the user explicitly asks for that separate engineering task.

Use the listed API paths through `aivax_invoke_function`. Search AIVAX context before changing uncertain fields, especially enum-like values, integration payloads, audio settings, and scheduled-continuation options.

```text
aivax_invoke_function({
  "method": "GET",
  "path": "/api/v1/web-chat-client/<client-id>"
})

aivax_invoke_function({
  "method": "PUT",
  "path": "/api/v1/web-chat-client/<client-id>",
  "body": {
    "clientParameters": {
      "allowedFrameOrigins": ["https://example.com"]
    }
  }
})
```

## Discovery Workflow

```text
GET /api/v1/web-chat-client
GET /api/v1/web-chat-client/<client-id>
GET /api/v1/ai-gateways/<gateway-id>
GET /api/v1/web-chat-client/<client-id>/sessions
GET /api/v1/web-chat-client/<client-id>/sessions?filter=<tag-or-session-id>
GET /api/v1/conversations?offsetminutes=1440&filter=--chat-client <client-id>
GET /api/v1/conversations?offsetminutes=1440&filter=--chat-session <session-id>
```

## Create a Chat Client

Create the gateway first, then create the client:

```text
POST /api/v1/web-chat-client
{
  "name": "Support Chat",
  "aiGatewayId": "<gateway-id>",
  "limitingParameters": {
    "messagesPerHour": 60,
    "maxMessages": 300
  },
  "clientParameters": {
    "pageTitle": "Support Chat",
    "primaryColor": "#2379bf",
    "helloLabel": "Hi, how can I help?",
    "helloSubLabel": "Ask about your account, product, or order.",
    "textAreaPlaceholder": "Type your message",
    "allowedFrameOrigins": ["https://example.com"],
    "suggestionButtons": [
      { "label": "Order status", "prompt": "I want to check my order status." }
    ]
  }
}
```

For enum-like fields such as `inputModes` and audio source, prefer mirroring the representation returned by `GET /api/v1/web-chat-client/<id>` unless the API documentation gives a more specific payload shape. For scheduled continuations, upload modes, audio synthesis, and messaging integration settings, do not guess field shapes.

## Edit a Chat Client

Use `PUT /api/v1/web-chat-client/<id>` with only the fields to change. `clientParameters` and `limitingParameters` are shallow-merged:

```text
PUT /api/v1/web-chat-client/<client-id>
{
  "clientParameters": {
    "showToolCalls": true,
    "uploadUnsupportedFiles": true,
    "allowedFrameOrigins": ["https://console.example.com"]
  }
}
```

Common client parameters:

- `primaryColor`, `pageTitle`, `logoImageUrl`: visual identity.
- `helloLabel`, `helloSubLabel`, `textAreaPlaceholder`: first-message experience.
- `suggestionButtons`: high-value starting prompts.
- `allowedFrameOrigins`: domains allowed to embed the chat iframe.
- `inputModes`: image, document, and audio upload support.
- `showToolCalls`: expose tool-call activity in the UI.
- `debug`: browser console diagnostics; disable after troubleshooting.
- `uploadUnsupportedFiles`: allow generic file uploads.
- `audioSynthesisSource`, `audioSynthesisVoice`, `audioSynthesisInstruction`: text-to-speech.
- `summarizeTextBeforeAudioSynthesis`: reduce long audio replies.
- `allowScheduledContinuations`, `splitAnswerIntoMessageChunks`, `maxScheduledIgnoredZone`, `messageDebounceInterval`: messaging-channel behavior.

Common limiting parameters:

- `messagesPerHour`: per-session hourly message cap.
- `maxMessages`: message count retained in session context.

## Sessions

Create or refresh a session:

```text
POST /api/v1/web-chat-client/<client-id>/sessions
{
  "tag": "customer-42",
  "expires": 3600,
  "extraContext": "Customer account context...",
  "contextLocation": "https://example.com/account/42",
  "metadata": {
    "channel": "web",
    "tenant": "acme"
  }
}
```

If a non-expired session with the same `tag` exists, AIVAX refreshes it and returns the existing session. The response includes `sessionId`, `accessKey`, and `talkUrl`.

Use session tags for stable external users or business records. Avoid putting secrets or sensitive personal details in `extraContext`.

Treat `accessKey`, `talkUrl`, `contextLocation`, external user tags, session metadata, and channel identifiers as sensitive operational data. Do not paste them in final responses unless the user needs the exact value to continue the task.

## Messaging Integrations

Supported integration types are `Zapi`, `Telegram`, `EvolutionApi`, and `Kapso`.

View current integration settings through the chat client detail endpoint, then patch only the target integration:

```text
PUT /api/v1/web-chat-client/<client-id>/integrations
{
  "integrationType": "Telegram",
  "integrations": {
    "telegramIntegration": {
      "botToken": "<secret>",
      "sessionDuration": "03:00:00"
    }
  }
}
```

Remove one integration only when clearly requested:

```text
DELETE /api/v1/web-chat-client/<client-id>/integrations/Telegram
```

Keep integration secrets out of final responses and logs.

## Diagnostics

For production chat issues:

1. View the chat client and linked gateway.
2. List sessions and identify affected `tag`, `sessionId`, and token/message counts.
3. List conversations filtered by chat client or session.
4. View the problematic conversation.
5. Decide whether the fix belongs to client limits, session context, gateway instructions, RAG, tools, or integration parameters.

## Validation

After creation or edits, verify the client listing/detail, session creation or refresh, talk URL availability without exposing the key, recent conversations, allowed origins, upload modes, integration status, rate limits, and secret handling.

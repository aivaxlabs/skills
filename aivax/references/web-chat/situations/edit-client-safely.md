# Edit Web Chat Client Safely

Use when the agent must update a client parameter, a rate limit, or a messaging integration without replacing unrelated configuration.

## Objective

Apply one intentional change, preserve existing client behavior, and verify it through the affected user journey.

## Preconditions

- The client exists and its current detail has been captured.
- The exact field or integration to change is known.
- Customer-facing, cost-affecting, or destructive changes are approved.

## Decision Tree

1. Is the target a client parameter, a limit, or an integration? Use the dedicated endpoint for integrations.
2. Does the field widen access or increase cost? Uploads, message limits, and scheduled continuations require deliberate review.
3. Is the field a nested object or array? Confirm merge behavior with `aivax_search_context`; preserve all intended existing array members.
4. Does the configuration include a secret? Do not echo or report bot tokens, `hookKey` values, or credentials unless changing them intentionally.
5. Is removing an integration requested? Confirm the exact type because `DELETE` cannot be reversed.
6. Is a webhook URL leaked or still on the legacy route without `<hook-key>`? Rotate the key, then update the provider with the keyed URL before 2026-12-01. Rotation breaks the previous URL immediately.

## Construction

```text
GET /api/v1/web-chat-client/<client-id>
PUT /api/v1/web-chat-client/<client-id>
{
  "clientParameters": {
    "showToolCalls": true,
    "suggestionButtons": [
      { "label": "Order status", "prompt": "I want to check my order status." }
    ]
  }
}
GET /api/v1/web-chat-client/<client-id>
```

`clientParameters` and `limitingParameters` are shallow-merged. Preserve the full intended value for any array you update; do not send a partial conceptual diff.

For integrations, inspect the client first and then submit only the target integration structure:

```text
PUT /api/v1/web-chat-client/<client-id>/integrations
{
  "integrationType": "Telegram",
  "integrations": {
    "telegramIntegration": {
      "botToken": "<secret>",
      "sessionDuration": 10800
    }
  }
}
```

The update keeps the existing `hookKey`, or generates one for a new integration. Rotate it only with approval, because the provider must be reconfigured:

```text
POST /api/v1/web-chat-client/<client-id>/integrations/Telegram/hook-key
POST /api/v1/intl/tg/<client-id>/update
```

For Telegram, `update` registers the keyed webhook URL and its `secret_token`. For Zapi, EvolutionApi, and Kapso, set the keyed URL in the provider dashboard. See the webhook URL formats in `SKILL.md` (Integration Webhooks).

Remove an integration only with explicit approval:

```text
DELETE /api/v1/web-chat-client/<client-id>/integrations/Telegram
```

## Validation

- The client view shows the exact intended change.
- Unrelated limits, parameters, and integrations remain unchanged.
- The embed, upload, audio, or messaging flow affected by the change still works.
- `debug` is disabled after troubleshooting.
- No access key, bot token, hook key, or integration secret appears in logs or final reporting.
- After a hook-key rotation, a test message through the provider reaches the client; a 401 means the provider still uses the old URL or, for Telegram, `update` was not called.

## Failure Modes

- API rejects a field: verify its type and allowed enum with `aivax_search_context`.
- An array parameter (such as `suggestionButtons`) was accidentally narrowed or broadened: restore the captured intended array.
- An integration fails: confirm the supported payload from the live API reference and inspect the client detail; do not guess credentials or nested fields.
- Kapso webhooks return 401 after setting `webhookSecretKey`: the secret must match the one configured in Kapso, which signs each request with `X-Webhook-Signature`.
- A webhook returns 410: it still uses the legacy URL without `<hook-key>`; configure the keyed URL.
- A change affects response quality: inspect conversations and load `references/ai-gateways/`, `references/rag/`, or `references/text-inference/` as evidence indicates.

## Escalation

- Need full request tracing: load `references/observability/`.
- Need to rotate a leaked integration secret: load `references/account/situations/create-and-rotate-keys.md`.
- Need resilience diagnosis: load `references/resilience/`.

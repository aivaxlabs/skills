# Create Web Chat Client

Use when the agent must create a browser-facing AIVAX chat client linked to an existing AI Gateway.

## Objective

Create the smallest usable client with a deliberate embedding policy, bounded limits, and an interaction design that reflects the gateway's purpose.

## Preconditions

- The target gateway exists and is appropriate for user-facing traffic.
- The intended client name, embedding origin, and user-facing purpose are known.
- The user has approved the customer-facing configuration when it will go live.

## Decision Tree

1. Does a matching client already exist? List clients and inspect candidates before creating a duplicate.
2. Is the chat embedded? If yes, add only the exact trusted origin to `allowedFrameOrigins`.
3. Which media types are necessary? Enable only the required `inputModes`.
4. Is the conversation high-volume or public? Set conservative `messagesPerHour` and `maxMessages`, then revisit from real usage data.
5. Does the client need messaging integrations or audio synthesis? Create the base client first; configure unfamiliar optional settings only after checking the API reference.

## Construction

```text
GET /api/v1/ai-gateways/<gateway-id>
GET /api/v1/web-chat-client
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
    "helloLabel": "How can we help?",
    "helloSubLabel": "Ask about your account, product, or order.",
    "textAreaPlaceholder": "Type your message",
    "allowedFrameOrigins": ["https://example.com"],
    "suggestionButtons": [
      { "label": "Order status", "prompt": "I want to check my order status." }
    ]
  }
}
GET /api/v1/web-chat-client/<client-id>
```

Use a valid CSS color for `primaryColor`. Do not use wildcard origins or add development origins to production without explicit user direction.

## Validation

- The new client points to the intended gateway.
- The allowed origin is exact and no broader than required.
- Limits, welcome text, and suggestion prompts match the expected audience.
- The returned client detail reflects the submitted configuration.
- A browser smoke test can reach the intended chat surface when a usable embed or talk URL is available.

## Failure Modes

- Gateway is missing or unsuitable: inspect it through `references/ai-gateways/` before creating the client.
- An enum or optional configuration is rejected: use `aivax_search_context`; do not guess the payload.
- The client is embedded nowhere: confirm the intended host rather than allowing arbitrary origins.
- The client works but answers poorly: trace a representative conversation with `references/observability/` and tune the gateway or RAG, not superficial client copy.

## Escalation

- Need a session for a specific customer or page: load `situations/manage-session.md`.
- Need messaging integration or a production setting change: load `situations/edit-client-safely.md`.
- Need gateway configuration: load `references/ai-gateways/`.

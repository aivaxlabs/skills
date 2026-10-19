---
name: aivax-account
description: Operate AIVAX account-level resources — API keys, account state, plan, subscription and invoices, privacy settings, salt, secret hygiene. Load when the agent must create or rotate keys, inspect account or plan, change or cancel a subscription, check invoices or saved payment methods, change account parameters such as zero data retention, manage the hook salt, or audit secret handling.
---

# Account

This sub-skill owns the account-level resources on AIVAX. The agent uses these resources through `aivax_invoke_function` (MCP) or direct HTTPS calls. The discipline is treating credentials as sensitive, rotating them carefully, and inspecting the account state before any change.

## Operating Files

- `situations/create-and-rotate-keys.md`: create API keys, rotate them safely, and delete old keys.
- `situations/inspect-balance-and-usage.md`: read balance, usage, invoices, and plan state to inform a decision.
- `situations/manage-subscription.md`: read subscription state, change or cancel a plan, and handle invoices and saved payment methods.
- `situations/secret-hygiene.md`: keep API keys, hook salts, and provider keys out of logs and final responses.

## Key Types

AIVAX has two key families because browser-facing and server-side use cases have different risk profiles.

- **Private key** (`sk-aiv-acc`): server-side integrations and administrative API calls. Authenticated account APIs and OpenAI-compatible inference.
- **Public key** (`pk-aiv-`): restricted client-side calls to explicitly public routes (RAG query, RAG answer, speech generation, speech-to-text transcription, image generation, media descriptions, and chat completions with stripped tool surfaces).

When the agent writes a server-side integration, use a private key. When the agent writes a browser or widget experience, use a public key and keep the workflow limited to routes designed for public access.

## Endpoints

- `GET /api/v1/accounts/api-key`: list masked keys and the current account `salt`.
- `POST /api/v1/accounts/api-key`: create a key with a label, expiration, and type.
- `DELETE /api/v1/accounts/api-key/<key-id>`: delete a key.
- `GET /api/v1/information/balance`: get the current balance, plan, storage usage, and subscription allowance consumption.
- `GET /api/v1/information/usage?timeStart=<iso>&timeEnd=<iso>&dayCount=<n>`: get usage details; use the API parameters to select the period and listing size.
- `GET /api/v1/information/usage/monthly-report?month=YYYY-MM`: get monthly insights and top-cost model, resource, and SKU tables.
- `PUT /api/v1/accounts`: replace the account name and the whole `parameters` object (`allowConversationsLogging`, `automaticJsonHealing`, `allowSemanticDataCollection`, `zeroDataRetentionOnly`, `notifications`). Omitted settings revert to their defaults, so send every setting that must be kept. No documented account endpoint returns the current `parameters`; confirm the values to keep with the user before replacing. Administrator-managed account data is stored separately and is never changed by this call.
- `GET /api/v1/subscriptions`: subscription state and purchasable plans. `POST /api/v1/subscriptions/plan`, `POST /api/v1/subscriptions/cancel`, `POST /api/v1/subscriptions/resubscribe`: change, cancel, or undo a pending change. `GET /api/v1/subscriptions/payment-methods` and `POST /api/v1/subscriptions/payment-methods/portal`: saved cards and a short-lived management link. See `situations/manage-subscription.md`.
- `POST /api/v1/accounts/roll-salt`: roll the hook salt. Destructive: invalidates existing worker and integration validation secrets.

## Plan And Administrator-Managed Data

- Use `Custom` as the current plan name in balance responses and `X-Authenticated-Account-Plan`. Update integrations that still compare against the former `Reseller` name.
- Do not infer unlimited rate limits from `Custom`; its rate limits are administrator-configured. Consult [Pricing](https://docs.aivax.net/docs/pricing) and [Plans and limits](https://docs.aivax.net/docs/limits), and ask the administrator about account-specific terms.
- Legal contact information, the negotiated agreement, negotiated pricing, billing start, and pending plan changes are administrator-managed. They are not part of `parameters`, cannot be changed through `PUT /api/v1/accounts`, and are not exposed as editable fields. Route correction requests to an authorized administrator; do not guess administrative endpoints or permissions.
- Treat negotiated terms and legal identity data as confidential. Do not copy them into logs, shared skills, or unrelated reports.
- `Custom` plans are managed by the AIVAX team; the subscription endpoints reject plan changes for them.

## Privacy Settings

- `parameters.allowConversationsLogging` (default on): stores conversation records and RAG transaction telemetry. Turning it off removes the evidence used by `references/observability/`.
- `parameters.zeroDataRetentionOnly` (default off): restricts integrated text inference to providers whose catalog `dataCollection` is `Zero`. A model without such a provider fails with a client error before inference and without charge; choose another model or turn the setting off. It is enforced on integrated-model gateway and chat-completion routing; BYOK gateways and other generation endpoints are not documented as covered.
- A chat completion can also send `allow_logging: false` to skip the conversation record for that call. The account setting still applies.
- Changing any of these affects availability or audit evidence: require explicit user approval, and verify with a test call.

## Public Key Restrictions

Public keys can call only the following routes:

- RAG semantic search (`/api/v1/query`).
- RAG answer generation (`/api/v1/answer`).
- Speech generation (`/api/v1/generations/speech`).
- Speech-to-text transcription (`POST /api/v1/generations/audio/transcriptions`).
- Media descriptions (`/api/v1/generations/descriptions`).
- Image generation (`/api/v1/generations/images`).
- Chat completions with stripped tool surfaces (no MCP sources, no protocol functions, no built-in tools, no Bash, no skills, no sentinel options).

When a public key calls chat completions, the `model` must be a full AI Gateway UUID. Direct integrated-model calls and gateway slug lookup are disabled.

## Hook Salt

AIVAX can authenticate outbound requests to your services (workers, protocol function callbacks). The `X-Request-Nonce` header is a BCrypt hash derived from the account hook salt. Rolling the salt invalidates existing worker and integration validation secrets; only do this when the salt is suspected to be compromised.

## Cost Awareness

Balance, usage, and plan drive the cost of every operation. Inspect them before starting a large job, a model swap, or a batch run. Do not copy commercial figures or account-specific terms into skills; consult the current Pricing and Plans and limits pages.

## Validation

- The key list reflects the new key.
- The new key works end-to-end before the old key is deleted.
- The balance and usage views are accessible.
- After a plan change, `GET /api/v1/subscriptions` and `GET /api/v1/information/balance` show the expected `plan`.
- After a `PUT /api/v1/accounts`, a test call confirms the intended behavior (for example, a zero data retention restriction).
- The salt roll is reflected in the next hook validation.

## Limits

- A key with a negative duration does not expire. Use this carefully.
- Expired keys are rejected by authentication and are later removed by cleanup jobs.
- The salt roll is irreversible. There is no "un-roll" endpoint.

## Escalation

- A key is compromised: load `situations/create-and-rotate-keys.md` and rotate immediately.
- A secret leaked in a log or final response: load `situations/secret-hygiene.md` and rotate the affected credentials.
- Balance is low: load `references/cost-monitoring/situations/optimize-spend.md`. Prepaid credit top-ups and payment-provider details are not part of the documented agent API surface: send the user to the account console, and see Pricing for amounts and fees. Never create payment invoices on the user's behalf.
- A subscription renewal failed or an invoice is open: load `situations/manage-subscription.md`.
- The salt must be rolled: load `situations/create-and-rotate-keys.md` (the salt section) and confirm approval.
- The request fails with 429 or a transient error: load `references/resilience/` and classify before retrying.

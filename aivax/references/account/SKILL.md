---
name: aivax-account
description: Operate AIVAX account-level resources — API keys, account state, plan, salt, secret hygiene. Load when the agent must create or rotate keys, inspect account or plan, manage the hook salt, or audit secret handling.
---

# Account

This sub-skill owns the account-level resources on AIVAX. The agent uses these resources through `aivax_invoke_function` (MCP) or direct HTTPS calls. The discipline is treating credentials as sensitive, rotating them carefully, and inspecting the account state before any change.

## Operating Files

- `situations/create-and-rotate-keys.md`: create API keys, rotate them safely, and delete old keys.
- `situations/inspect-balance-and-usage.md`: read balance, usage, and plan state to inform a decision.
- `situations/secret-hygiene.md`: keep API keys, hook salts, and provider keys out of logs and final responses.

## Key Types

AIVAX has two key families because browser-facing and server-side use cases have different risk profiles.

- **Private key** (`sk-aiv-acc`): server-side integrations and administrative API calls. Authenticated account APIs and OpenAI-compatible inference.
- **Public key** (`pk-aiv-`): restricted client-side calls to explicitly public routes (RAG query, RAG answer, speech, image generation, media descriptions, and chat completions with stripped tool surfaces).

When the agent writes a server-side integration, use a private key. When the agent writes a browser or widget experience, use a public key and keep the workflow limited to routes designed for public access.

## Endpoints

- `GET /api/v1/keys`: list keys (masked).
- `POST /api/v1/keys`: create a key with a label, expiration, and type.
- `DELETE /api/v1/keys/<key-id>`: delete a key.
- `GET /api/v1/information/balance`: get the current balance, plan, storage usage, and reserve consumption.
- `GET /api/v1/information/usage?timeStart=<iso>&timeEnd=<iso>`: get usage details for a period.
- `PUT /api/v1/account`: update account metadata.
- `POST /api/v1/account/salt/roll`: roll the hook salt. Destructive: invalidates existing worker and integration validation secrets.

## Public Key Restrictions

Public keys can call only the following routes:

- RAG semantic search (`/api/v1/query`).
- RAG answer generation (`/api/v1/answer`).
- Speech generation (`/api/v1/generations/speech`).
- Media descriptions (`/api/v1/generations/describe-media`).
- Image generation (`/api/v1/generations/images`).
- Chat completions with stripped tool surfaces (no MCP sources, no protocol functions, no built-in tools, no Bash, no skills, no sentinel options).

When a public key calls chat completions, the `model` must be a full AI Gateway UUID. Direct integrated-model calls and gateway slug lookup are disabled.

## Hook Salt

AIVAX can authenticate outbound requests to your services (workers, protocol function callbacks). The `X-Request-Nonce` header is a BCrypt hash derived from the account hook salt. Rolling the salt invalidates existing worker and integration validation secrets; only do this when the salt is suspected to be compromised.

## Cost Awareness

Balance, usage, and plan drive the cost of every operation. Inspect them before starting a large job, a model swap, or a batch run. The Free plan has a context cap and a commission multiplier of 1.25x. The Pro plan has 1.05x. The Max plan has 1.00x.

## Validation

- The key list reflects the new key.
- The new key works end-to-end before the old key is deleted.
- The balance and usage views are accessible.
- The salt roll is reflected in the next hook validation.

## Limits

- A key with a negative duration does not expire. Use this carefully.
- Expired keys are rejected by authentication and are later removed by cleanup jobs.
- The salt roll is irreversible. There is no "un-roll" endpoint.

## Escalation

- A key is compromised: load `situations/create-and-rotate-keys.md` and rotate immediately.
- A secret leaked in a log or final response: load `situations/secret-hygiene.md` and rotate the affected credentials.
- Balance is low: load `references/cost-monitoring/situations/optimize-spend.md`.
- The salt must be rolled: load `situations/create-and-rotate-keys.md` (the salt section) and confirm approval.

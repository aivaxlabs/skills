# Secret Hygiene

Use when the agent must handle AIVAX credentials — API keys, hook salts, provider keys, integration tokens, access keys — without leaking them in logs, final responses, or persistent storage.

## Objective

Keep credentials out of every place they should not be: console output, log files, error messages, exported conversation records, `metadata` fields, and final responses to the user.

## Preconditions

- The agent has or generates credentials during the task.
- The user is aware that credentials are sensitive and should not be shared.

## Decision Tree

1. Is the credential a private API key (`sk-aiv-acc`)? Treat it as a password. Never echo it back.
2. Is the credential a public API key (`pk-aiv-`)? Still sensitive — it can call public routes and consume balance. Echo only when the user needs the literal value.
3. Is the credential a hook salt or hook key? The salt is used to derive nonces; the hook key validates them. Both are sensitive.
4. Is the credential a provider key for an external BYOK model? Sensitive. Preserve when patching unrelated gateway fields.
5. Is the credential an integration token (Telegram bot, WhatsApp, etc.)? Sensitive. Never echo.
6. Is the credential a chat client access key or talk URL? Sensitive. Echo only when the user needs the literal value to continue the task.
7. Is the credential an external user ID, session ID, or session metadata? Sensitive in the sense that it is operational data; treat with care.

## Rules

- Never print the full key. The key list endpoint returns masked values; trust the mask and do not re-fetch the full value unless the application needs it.
- Never store the key in `metadata`, in the conversation record, in the trace, or in any field that is exported.
- Never commit the key to a repository. If the agent writes code, use a secret manager and a placeholder.
- Never paste the key in a final response unless the user explicitly needs the literal value to continue the task. When the user does need it, prefer a UI surface that the user can copy from, not a chat reply.
- When logging, redact the key. A safe pattern is to log the prefix (`sk-aiv-acc-...`) and the last 4 characters, never the middle.
- When the key is compromised, rotate immediately. Load `situations/create-and-rotate-keys.md`.

## Validation

- The full key is not in any log, error message, or final response.
- The key is stored in a secret manager or environment variable, not in code.
- The trace and the conversation record do not contain the key.
- The user has confirmed the value when the user needs it for the task.

## Failure Modes

- The agent echoes the key in a log: rotate the key, update the secret store, and re-run.
- The agent commits the key to a repository: rotate the key, remove the commit, and update the secret store.
- The agent writes the key in a final response: rotate the key, redact the response, and update the workflow to avoid the pattern.

## Escalation

- A key has been compromised: rotate immediately. Load `situations/create-and-rotate-keys.md`.
- The salt must be rolled: confirm the impact on workers and integrations before rolling. Load `situations/create-and-rotate-keys.md` (the salt section).
- A provider key was leaked in a gateway patch: load `references/ai-gateways/situations/edit-gateway-safely.md` and rotate the provider key.

# Create And Rotate Keys

Use when the agent must create a new API key, rotate an existing one, or delete a key.

## Objective

Keep credentials fresh, scoped, and auditable, without disrupting running services.

## Preconditions

- The account is configured.
- The agent knows the intended use of the key (server-side vs. client-side, scope, lifetime).
- The agent has confirmed with the user when the change is destructive (deleting a key, rolling the salt).

## Decision Tree

1. What is the key's intended use?
   - Server-side, administrative: private key (`sk-aiv-acc`).
   - Browser or widget, public routes: public key (`pk-aiv-`).
2. What is the lifetime? A negative duration means the key never expires. A positive duration sets the expiration.
3. What is the label? A short, descriptive label that names the application or the user. Useful for rotation and audit.
4. Is the rotation triggered by a compromise, a policy, or a planned refresh?
   - Compromise: rotate immediately. Delete the old key after the new one is validated.
   - Policy: rotate on a schedule. Create the new key, validate it, and schedule the deletion of the old key.
   - Planned refresh: create the new key, validate it, and delete the old key in the same change.

## Construction

```text
POST /api/v1/keys
{
  "label": "<application-or-user>",
  "type": "private" | "public",
  "expiresIn": "<duration-or-negative>"
}
```

The response includes the full key. Store it in a secret manager; do not echo it back in any log or final response.

## Rotation Pattern

1. Create the new key with the same label and a clear version (e.g. `app-prod-v3`).
2. Update the application's secret store with the new key.
3. Validate the new key end-to-end with a smoke test.
4. Delete the old key only after the new key is validated.
5. Record the rotation in the change log.

## Salt Roll

The salt roll is a separate, destructive operation. Use it only when the salt is suspected to be compromised.

```text
POST /api/v1/account/salt/roll
```

The operation is irreversible. All existing worker and integration validation secrets are invalidated. Coordinate the roll with the services that validate the nonce.

## Validation

- The key list reflects the new key.
- The new key works end-to-end before the old key is deleted.
- The salt roll is reflected in the next hook validation.
- The change is recorded in the change log.

## Failure Modes

- The new key does not work: the type is wrong, the expiration is too short, or the label is invalid. Inspect the error.
- The old key is deleted before the new key is validated: the application is locked out. Restore from the secret manager backup.
- The salt roll is partial: services that depend on the old nonce fail. Coordinate the roll.

## Escalation

- A key is compromised: rotate immediately. Do not wait for the next maintenance window.
- The salt must be rolled: confirm the impact on workers and integrations before rolling.
- The balance is low and a rotation would consume balance: load `references/cost-monitoring/situations/optimize-spend.md`.

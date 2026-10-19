---
name: aivax-notifications
description: Read AIVAX account notifications and the unread count. Load when the agent must check account alerts such as balance warnings, list notifications newest-first, or count unread notifications.
---

# Notifications

This sub-skill owns access to AIVAX account notifications. Notifications are account-level alerts returned as minimal summaries ordered from newest to oldest. Listing notifications has a read-state side effect.

## Operating Files

- `situations/triage-notifications.md`: check alerts while preserving the pre-list unread count.

## When To Use Notifications

Use this sub-skill to:

- Check account alerts such as low-balance warnings.
- List notifications for monitoring or status checks.
- Count unread notifications before deciding whether anything needs attention.

Do not use this sub-skill to investigate spend behind an alert or to audit conversation history. Those causes belong to other skills (see Escalation).

## Endpoints

- `GET /api/v1/notifications`: list notification summaries, newest first, then mark all account notifications as read. Each returned item contains `id`, `createdAt`, `type`, `subject`, `message`, and `readAt` (`readAt` is null when the item was unread at listing time).
- `GET /api/v1/notifications/unread/count`: return `{"count": N}` with the number of unread notifications without marking them as read.

## Notification Types

The `type` field is a string and new types can appear; do not hard-code an exhaustive list. Types produced by the platform include `LowBalanceNotification`, `DeprecatedModelsNotification`, `FreeAccountInactivityNotification`, `WeeklyTestingSummary`, `SubscriptionRenewalFailureNotification`, and `AdministrativeNotification`. Which preferences generate them (low balance, deprecated models, weekly testing summary, repeated test failures) are `parameters.notifications` settings, edited through `references/account/`.

## Validation

1. Call `GET /api/v1/notifications/unread/count` first when the pre-list unread count is needed.
2. Then call `GET /api/v1/notifications` and verify the list is ordered newest-first and each item carries the documented fields.
3. Do not validate the pre-list count against a new count after listing: the list operation marks notifications as read.

## Limits

- This surface exposes no separate delete, dismiss, or mark-as-read endpoint. Do not invent one; listing itself marks all account notifications as read.
- Only the minimal summary fields are returned; full delivery or audit history is not available here.

## Escalation

- A `LowBalanceNotification` needs spend analysis: load `references/cost-monitoring/`.
- A `SubscriptionRenewalFailureNotification` means a subscription invoice is unpaid and the account may return to `Free`: load `references/account/situations/manage-subscription.md`. Do not pay or retry the charge for the user.
- A `DeprecatedModelsNotification` means a gateway or workflow may depend on a retiring model: load `references/ai-gateways/` or `references/text-inference/situations/choose-model.md`.
- The alert points to a production failure or needs an audit trail: load `references/observability/`.

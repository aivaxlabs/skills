---
name: aivax-notifications
description: Read AIVAX account notifications and the unread count. Load when the agent must check account alerts such as balance warnings, list notifications newest-first, or count unread notifications.
---

# Notifications

This sub-skill owns access to AIVAX account notifications. Notifications are account-level alerts returned as minimal summaries ordered from newest to oldest. Listing notifications has a read-state side effect.

## When To Use Notifications

Use this sub-skill to:

- Check account alerts such as low-balance warnings.
- List notifications for monitoring or status checks.
- Count unread notifications before deciding whether anything needs attention.

Do not use this sub-skill to investigate spend behind an alert or to audit conversation history. Those causes belong to other skills (see Escalation).

## Endpoints

- `GET /api/v1/notifications`: list notification summaries, newest first, then mark all account notifications as read. Each returned item contains `id`, `createdAt`, `type`, `subject`, `message`, and `readAt` (`readAt` is null when the item was unread at listing time).
- `GET /api/v1/notifications/unread/count`: return `{"count": N}` with the number of unread notifications without marking them as read.

## Validation

1. Call `GET /api/v1/notifications/unread/count` first when the pre-list unread count is needed.
2. Then call `GET /api/v1/notifications` and verify the list is ordered newest-first and each item carries the documented fields.
3. Do not validate the pre-list count against a new count after listing: the list operation marks notifications as read.

## Limits

- This surface exposes no separate delete, dismiss, or mark-as-read endpoint. Do not invent one; listing itself marks all account notifications as read.
- Only the minimal summary fields are returned; full delivery or audit history is not available here.

## Escalation

- A `LowBalanceNotification` needs spend analysis: load `references/cost-monitoring/`.
- The alert points to a production failure or needs an audit trail: load `references/observability/`.

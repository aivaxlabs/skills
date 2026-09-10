---
name: aivax-notifications
description: Read AIVAX account notifications and the unread count. Load when the agent must check account alerts such as balance warnings, list notifications newest-first, or count unread notifications.
---

# Notifications

This sub-skill owns read-only access to AIVAX account notifications. Notifications are account-level alerts returned as minimal summaries ordered from newest to oldest. This sub-skill performs no mutations.

## When To Use Notifications

Use this sub-skill to:

- Check account alerts such as low-balance warnings.
- List notifications for monitoring or status checks.
- Count unread notifications before deciding whether anything needs attention.

Do not use this sub-skill to investigate spend behind an alert or to audit conversation history. Those causes belong to other skills (see Escalation).

## Endpoints

- `GET /api/v1/notifications`: list notification summaries, newest first. Each item contains `id`, `createdAt`, `type`, `subject`, `message`, and `readAt` (`readAt` is null when unread).
- `GET /api/v1/notifications/unread/count`: return `{"count": N}` with the number of unread notifications.

## Validation

- The list is ordered newest-first.
- Each item carries the documented fields; treat `readAt: null` as unread.
- The unread count is consistent with the number of listed items with `readAt: null` in the same read window.
- The trace ID is preserved.

## Limits

- Read-only: this surface exposes no delete, dismiss, or mark-as-read operation. Do not invent one.
- Only the minimal summary fields are returned; full delivery or audit history is not available here.

## Escalation

- A `LowBalanceNotification` needs spend analysis: load `references/cost-monitoring/`.
- The alert points to a production failure or needs an audit trail: load `references/observability/`.

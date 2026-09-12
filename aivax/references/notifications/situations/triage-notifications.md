# Triage Notifications

Use when the agent must check account alerts without losing the unread count to the list side effect.

## Objective

Report the current alerts newest-first while preserving an accurate pre-list unread count for the caller.

## Preconditions

- The caller needs either the unread count, the alert contents, or both.

## Decision Tree

1. Is the pre-list unread count needed? Call `GET /api/v1/notifications/unread/count` first; it does not mark anything as read.
2. List with `GET /api/v1/notifications` only when the alert contents are actually needed. Listing marks all account notifications as read.
3. Does an alert need follow-up? Escalate by type: spend behind a `LowBalanceNotification` goes to cost-monitoring; production failure or audit trail goes to observability.

## Construction

```text
GET /api/v1/notifications/unread/count   (no side effect)
GET /api/v1/notifications                (marks all as read; newest first)
```

Each listed item carries `id`, `createdAt`, `type`, `subject`, `message`, and `readAt` (`readAt` is null when the item was unread at listing time).

## Validation

- The count is captured before any listing call.
- The list is ordered newest-first with the documented fields.
- No post-list count is compared against the pre-list count; the list operation invalidates that comparison.

## Failure Modes

- Listing before counting: the unread count is already lost. Count first on the next check.
- Treating listing as non-destructive: there is no separate dismiss endpoint; listing itself is the mark-as-read.

## Limits

- Only minimal summary fields are returned; full delivery or audit history is not available here.

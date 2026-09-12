# Review and Cancel a Schedule

Use when the agent must find a one-time schedule or recurring task for a web chat client and cancel the one the user no longer wants.

## Objective

Resolve the exact item under the owning web chat client, confirm it with a get, and cancel it with explicit user approval.

## Preconditions

- The owning `<web-chat-id>` is known.
- The target is identified by user, time window, reason, or cron pattern — never by a guessed ID.

## Decision Tree

1. One-time schedule or recurring task? List with `GET .../hook-schedules` (optionally `?external-user-id=` to scope to one user) and read the derived `state` (`Pending`, `Activated`, `Failed`) plus `cronExpression`, `nextActivation`, and `executionCount` for recurring tasks.
2. Confirm the candidate with the matching get: `GET .../schedules/<id>` or `GET .../recurring/<id>`. A 404 means a wrong ID or a wrong client, not a system error.
3. Cancel only with explicit user approval: `DELETE .../schedules/<id>` or `DELETE .../recurring/<id>`. Cancellation is irreversible.
4. Confirm the item no longer appears in the list response.

## Construction

```text
GET /api/v1/web-chat-clients/<web-chat-id>/hook-schedules?external-user-id=<user>
GET /api/v1/web-chat-clients/<web-chat-id>/hook-schedules/schedules/<id>
GET /api/v1/web-chat-clients/<web-chat-id>/hook-schedules/recurring/<id>
DELETE /api/v1/web-chat-clients/<web-chat-id>/hook-schedules/schedules/<id>
DELETE /api/v1/web-chat-clients/<web-chat-id>/hook-schedules/recurring/<id>
```

## Validation

- The inspected item belongs to the `<web-chat-id>` in the route.
- The approval names the exact ID and the owning client before the delete.
- The post-cancel list no longer contains the item.
- The trace ID is preserved.

## Failure Modes

- Cancelling by guessed ID: always resolve from list or get first.
- Cancelling under the wrong client: the route is client-scoped; re-resolve under the right `<web-chat-id>`.

## Limits

- No creation endpoint exists on this module; creation happens through model gateway tools.
- The list `state` field is derived; the get operations return raw activation fields.

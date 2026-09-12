---
name: aivax-hook-schedules
description: Inspect and cancel scheduled hook activations for a web chat client — one-time schedules and recurring tasks. Load when the agent must list, inspect, or cancel a scheduled follow-up or recurring message owned by a web chat client.
---

# Hook Schedules

This sub-skill owns scheduled hook activations for a web chat client: one-time schedules and recurring tasks that deliver messages to external users. All routes live under `/api/v1/web-chat-clients/<web-chat-id>/hook-schedules`, where `<web-chat-id>` is the owning web chat client (see `references/web-chat/`).

There is no POST on this module: schedules are created by the model through gateway tools during a conversation, not through a direct API call. This sub-skill inspects and cancels; it never creates.

## Operating Files

- `situations/review-and-cancel.md`: resolve the exact schedule or recurring task and cancel it with approval.

## When To Use Hook Schedules

Use this sub-skill to:

- List pending, activated, or failed one-time schedules and recurring tasks for a web chat client.
- Inspect one schedule or recurring task before cancelling it.
- Cancel a schedule or recurring task the user no longer wants.
- Review recently inactive users: users who received a scheduled message in the last 7 days but did not reply since.

Do not use this sub-skill to create a schedule; creation happens through model gateway tools, not through this API. Do not use it to edit the chat client itself; use `references/web-chat/` for that.

## Endpoints

- `GET /api/v1/web-chat-clients/<web-chat-id>/hook-schedules`: list `webChat` (id, name), `inactiveUsers`, `schedules`, and `recurringTasks`. Accepts an optional `?external-user-id=` query parameter to restrict schedules and recurring tasks to one user.
- `GET .../hook-schedules/schedules/<id>`: return one one-time schedule (`scheduledTime`, `reason`, `context`, activation fields).
- `GET .../hook-schedules/recurring/<id>`: return one recurring task, including `cronExpression`, `executionCount`, `maxExecutions`, `lastActivation`, and `nextActivation`.
- `DELETE .../hook-schedules/schedules/<id>`: cancel one one-time schedule. Returns 404 when the schedule does not belong to the client.
- `DELETE .../hook-schedules/recurring/<id>`: cancel one recurring task. Returns 404 when the task does not belong to the client.

The list response marks each one-time schedule with a derived `state` of `Pending`, `Activated`, or `Failed`. The get operations return the raw activation fields instead.

## Safety

- Cancellation is destructive and irreversible. Confirm the schedule or task ID and its owning web chat client with a GET before deleting, and proceed only with explicit user approval.
- Never cancel by guessing an ID. Always resolve it from the list or get operation first.

## Validation

- The inspected schedule or task belongs to the `<web-chat-id>` in the route; a 404 means a wrong ID or a wrong client, not a system error.
- After a cancel, the item no longer appears in the list response.
- The trace ID is preserved.

## Limits

- No creation endpoint exists on this module; do not fabricate a POST payload.
- The list `state` field is derived; do not treat it as stored data.

## Escalation

- The owning chat client is misconfigured: load `references/web-chat/`.
- The user context behind a schedule is missing: use the memories generate/migration flows to rebuild it first.
- The cancel hits rate-limit or transient failures: load `references/resilience/`.

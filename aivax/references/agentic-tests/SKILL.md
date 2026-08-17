---
name: aivax-agentic-tests
description: Create, run, inspect, cancel, schedule, and compare AIVAX Agentic Tests using the implemented conversational simulator-and-judge schema. Covers persisted test definitions and runs, ephemeral SSE evaluations, exact request fields, thresholds, run states, results, conversations, usage, cost, and notifications.
---

# Agentic Tests

Use this skill for AIVAX Agentic Tests. A test runs one bounded simulated conversation against an AI gateway or integrated model. A simulated user follows the role and conversational flow described by `goal`; the target assistant responds; an internal judge evaluates the complete conversation against `goal` and `validation_criteria`. One definition can exercise several related situations and acceptance criteria when they fit one coherent conversation. It is not an isolated prompt/response assertion or an input dataset.

This API does not define or use an expected-output list, custom judge model, or separate post-run evaluation request. Unknown request properties may be silently ignored, so their presence does not indicate support.

Before an unfamiliar or account-changing call, follow `../platform-rules/SKILL.md`. Prefer `aivax_invoke_function` when available. Otherwise use the documented HTTP API with authentication. Do not invent fields that are not listed here; use `aivax_search_context` if live documentation may be newer than this contract.

## Operating Files

- `situations/design-test-case.md`: construct a valid conversational test definition.
- `situations/run-and-collect-traces.md`: queue or stream an evaluation and inspect its retained conversation.
- `situations/evaluate-by-criteria.md`: write judge-only criteria and interpret judge results.
- `situations/compare-runs.md`: compare repeated runs without assuming dataset metrics.

## Choose The Execution Mode

- **Persisted**: use `/api/v1/agentic-tests` to save a reusable definition, schedule it, queue runs, inspect history, and cancel pending or running work.
- **Ephemeral**: `POST /api/v1/generations/agentic-tests` streams one direct evaluation as server-sent events and does not create a test or retained run. `/api/v1/generations/validations` is a compatibility alias.

Both modes consume account balance. Persisted runs are asynchronous. The ephemeral endpoint requires positive operating balance and returns an SSE stream even when `stream` is omitted or false.

## Persisted Endpoints

| Method | Path | Purpose |
| --- | --- | --- |
| `GET` | `/api/v1/agentic-tests?search=&gateway=` | List owned tests with `latest_run`. |
| `POST` | `/api/v1/agentic-tests` | Create a definition; returns `201`. |
| `GET` | `/api/v1/agentic-tests/{test-id}` | Get the complete definition and latest run. |
| `PATCH` | `/api/v1/agentic-tests/{test-id}` | Partially update supplied fields. |
| `DELETE` | `/api/v1/agentic-tests/{test-id}` | Delete the test and retained runs; returns `204`. |
| `POST` | `/api/v1/agentic-tests/{test-id}/runs` | Queue a run from the current definition; no body; returns `202`. |
| `GET` | `/api/v1/agentic-tests/{test-id}/runs?state=&limit=&offset=` | List run summaries, newest first. |
| `GET` | `/api/v1/agentic-tests/{test-id}/runs/{run-id}` | Get result, conversation, usage, cost, and error. |
| `POST` | `/api/v1/agentic-tests/{test-id}/runs/{run-id}/cancel` | Cancel pending work immediately or request cooperative cancellation; no body. |

`state` accepts `pending`, `running`, `succeeded`, `failed`, or `cancelled`. `limit` defaults to 50 and is clamped to 1–100; `offset` defaults to 0.

## Persisted Definition Schema

`POST /api/v1/agentic-tests` accepts:

| Field | Type | Required/default | Contract |
| --- | --- | --- | --- |
| `name` | string | required | Non-empty; maximum 200 characters. |
| `gateway` | string | required | Available AI gateway slug or integrated model name; maximum 200 characters. |
| `goal` | string, message-part object, or message-part array | required | Shared with the simulated user and judge. |
| `validation_criteria` | string, message-part object, array, or null | `null` | Visible only to the judge. |
| `start` | message array | `[]` | Initial OpenAI-compatible conversation messages. |
| `resources` | external-resource array | `[]` | Up to 16 `{ "type", "data" }` objects that provide additional shared context to the simulated user and judge. Use `Text` for literal non-empty `data` or `RemoteResource` to retrieve content from the non-empty URL in `data`. Resources are not conversation messages and are not sent to the gateway under test as chat history. |
| `max_turns` | integer | `10` | 2–64 simulated user turns. |
| `minimum_turns` | integer | `1` | At least 1 and less than `max_turns`. |
| `allow_user_exit` | boolean | `true` | Allows the simulator to end after `minimum_turns`. |
| `judge_start_turn` | integer | `1` | At least 1 and less than `max_turns`. |
| `loss_threshold` | number | `0.2` | 0.01–0.99; below `base_threshold`. |
| `base_threshold` | number | `0.9` | 0.01–0.99; more than 0.1 above `loss_threshold`. |
| `profile` | string | `medium` | `low`, `medium`, or `high`; selects internal simulator and judge profiles, not a custom judge model. |
| `user_sampling.top_k` | number | `0.4` | 0–2. |
| `user_sampling.max_decay` | number | `0.02` | 0–1. |
| `metadata` | object | `{}` | Forwarded to gateway inference; use string values for compatibility with execution. |
| `external_user_id` | string or null | `null` | Forwarded to gateway inference; maximum 200 characters. |
| `cron` | string or null | `null` | Standard five-field cron; minimum interval is five minutes. |
| `enabled` | boolean | `true` | Enables scheduled execution; manual runs can still be queued. |
| `notification_threshold` | integer | `1` | Consecutive execution failures before notification; minimum 1. |
| `recovery_notification` | boolean | `true` | Notify when a successful run follows the configured failure streak. |

`PATCH` accepts any subset of the same fields. Omitted fields remain unchanged. Send explicit `null` to clear nullable fields such as `validation_criteria`, `external_user_id`, or `cron`.

```json
{
  "name": "Checkout regression",
  "gateway": "checkout-gateway",
  "goal": "You are shopping with the store assistant. You, the user, select two blue shirts, provide the information needed for checkout, and continue until the order is ready to be placed.",
  "validation_criteria": "- Validate whether the assistant states the final total before placing the order.\n- Validate whether the assistant asks for explicit confirmation before checkout.\n- Validate whether the assistant does not claim the order was placed before that confirmation.",
  "start": [],
  "resources": [
    {
      "type": "Text",
      "data": "The customer has a 14-day refund window."
    },
    {
      "type": "RemoteResource",
      "data": "https://example.com/refund-policy"
    }
  ],
  "max_turns": 10,
  "minimum_turns": 1,
  "allow_user_exit": true,
  "judge_start_turn": 1,
  "loss_threshold": 0.2,
  "base_threshold": 0.9,
  "profile": "medium",
  "user_sampling": {
    "top_k": 0.4,
    "max_decay": 0.02
  },
  "metadata": {},
  "external_user_id": null,
  "cron": null,
  "enabled": true,
  "notification_threshold": 1,
  "recovery_notification": true
}
```

### Resources

Use `resources` for stable facts, policy text, product reference material, or other context that the simulated user and judge must share but that should not be represented as a prior message. Each entry has a non-empty `data` field and a `type` of `Text` or `RemoteResource`; send no more than 16 entries.

`Text` embeds `data` literally. `RemoteResource` retrieves the URL in `data` during evaluation. Use only trusted, publicly reachable URLs with content appropriate for the test. Remote content can change between runs and adds processing usage, so prefer `Text` when the test needs a fixed, reproducible reference. Do not use a remote resource to provide the assistant's previous message: put that message in `start` with its original role instead.

## Ephemeral Request Schema

`POST /api/v1/generations/agentic-tests` uses the same simulation fields, except:

- use `model` instead of `name` and `gateway`;
- scheduling and notification fields are not used; the current implementation may silently ignore them and other unknown properties;
- `stream` is read for compatibility but the response is always SSE.

```json
{
  "model": "checkout-gateway",
  "goal": "You are shopping with the store assistant. You, the user, select two blue shirts, provide the information needed for checkout, and continue until the order is ready to be placed.",
  "validation_criteria": "- Validate whether the assistant states the final total before placing the order.\n- Validate whether the assistant asks for explicit confirmation before checkout.\n- Validate whether the assistant does not claim the order was placed before that confirmation.",
  "start": [],
  "resources": [
    {
      "type": "Text",
      "data": "The customer has a 14-day refund window."
    },
    {
      "type": "RemoteResource",
      "data": "https://example.com/refund-policy"
    }
  ],
  "max_turns": 10,
  "minimum_turns": 1,
  "allow_user_exit": true,
  "judge_start_turn": 1,
  "loss_threshold": 0.2,
  "base_threshold": 0.9,
  "profile": "medium",
  "user_sampling": {
    "top_k": 0.4,
    "max_decay": 0.02
  },
  "metadata": {},
  "external_user_id": "customer-123",
  "stream": true
}
```

## Response Contract

Normal JSON endpoints use the AIVAX response envelope; read their payload from `data`. A test detail contains all definition fields plus `id`, `next_run_at`, `consecutive_failures`, `last_notification_at`, `created_at`, `updated_at`, and `latest_run`.

A run summary contains:

```json
{
  "id": "019...",
  "test_id": "019...",
  "state": "succeeded",
  "created_at": "2026-08-16T03:00:00",
  "started_at": "2026-08-16T03:00:05",
  "finished_at": "2026-08-16T03:01:02",
  "score": 0.94,
  "cost": 0.0123,
  "error": null,
  "external_user_id": null
}
```

A run detail adds `test_name`, `result`, and `conversation`. Each conversation item has `timestamp`, role (`user`, `assistant`, or `judge`), `content`, and `usage` with `model`, `prompt_tokens`, `cached_prompt_tokens`, `completion_tokens`, and `cost`.

`state: succeeded` means execution completed without an operational error; it does not mean the goal passed. Determine behavioral outcome from `result.outcome` (`success`, `loss`, or `incomplete`) and inspect `result.reason`, `result.state`, `result.turn_number`, `result.conversation_delta`, and `result.loss_streak`. `score` is the final `conversation_delta`, not a dataset average.

## Run Lifecycle And Safety

1. Create or retrieve the test and verify the complete definition.
2. For a manual persisted run, call `POST .../{test-id}/runs` with no body.
3. Poll the run or filtered run list until `succeeded`, `failed`, or `cancelled`; use bounded waits and do not repeatedly queue duplicates.
4. Inspect the complete run. Separate execution state, behavioral outcome, retained judge reasoning, error, and cost.
5. Cancel only when the user requests it or an agreed limit is exceeded. Completed inference remains billable.

Scheduled runs follow plan concurrency: Free 1, Pro 4, Max/Reseller 8 concurrent runs per account. A run can fail before or during execution when balance is insufficient. Notifications track execution failures (`state: failed`), not low behavioral scores in otherwise succeeded runs.

## Test Authoring Rules

- Test the gateway that actually serves the behavior under review. Do not substitute an `eval-*` gateway merely because its name suggests evaluation; confirm the intended gateway from the live account or authoritative configuration.
- Treat the unit under test as a complete conversation. Group related situations and multiple acceptance criteria when the simulated user can encounter them in one coherent flow.
- Split definitions when they require incompatible user roles, mutually exclusive objectives, materially different opening contexts, or different production opening templates.
- Write `goal` as the simulated user's identity, context, intent, and likely conversational progression. Address the simulator directly, for example: `You are speaking with Maya, the insurer's assistant. You, the user, ...` Do not use `goal` as a list of instructions for the assistant.
- Write `validation_criteria` as observable assistant behavior. Use explicit checks such as `Validate whether the assistant...`, including required behavior, prohibited behavior, ordering constraints, and terminal behavior.
- Ground the target, opening messages, facts, workflows, and expected behavior in live configuration or authoritative product material. Do not invent an oracle where the product contract is unclear.
- For assistant-initiated or outbound flows, place the real configured assistant opening in `start`. Preserve its role and wording instead of paraphrasing it into `goal`.

## Important Non-Features

Do not send or claim support for `target`, `instruction`, `inputs`, `expected`, `criteria`, `runId`, `judgeModel`, `temperature`, or a persisted `/evaluate` endpoint. Several related situations may be covered by one evolving conversation, but they are not dataset rows. Repeat controlled runs when confidence across stochastic paths matters.

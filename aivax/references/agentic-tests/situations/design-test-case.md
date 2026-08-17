# Design A Test Case

Use when defining a persisted or ephemeral Agentic Test.

## Objective

Describe one realistic simulated conversation with a measurable goal and judge-only criteria, using only fields accepted by the API.

## Design Decisions

1. **Target**: choose an available AI gateway slug or integrated model name. Persisted requests use `gateway`; ephemeral requests use `model`. Chat-client IDs are not targets.
2. **Goal**: state what the simulated user must accomplish. The simulator and judge both see it. Do not write assistant instructions here.
3. **Validation criteria**: add observable constraints that only the judge should see. Keep them compatible with the goal and avoid hidden facts the conversation cannot establish.
4. **Initial conversation**: use `start` only when prior messages are materially part of the scenario. It is an array of OpenAI-compatible messages, not an input dataset.
5. **Turn budget**: `max_turns` is 2–64. `minimum_turns` and `judge_start_turn` are each at least 1 and less than `max_turns`.
6. **Thresholds**: `loss_threshold` and `base_threshold` are each 0.01–0.99; loss must be lower and the distance must be greater than 0.1.
7. **Profile**: `low`, `medium`, or `high` selects the platform's simulator and judge profile. There is no `judgeModel` field.
8. **Persistence**: add cron and notification fields only to persisted definitions.

## Minimal Persisted Request

```http
POST /api/v1/agentic-tests
Content-Type: application/json
```

```json
{
  "name": "Checkout confirms total",
  "gateway": "checkout-gateway",
  "goal": "Place an order for two blue shirts",
  "validation_criteria": "Success requires the assistant to state the final total and ask for confirmation before placing the order."
}
```

Defaults fill `start`, turn controls, thresholds, profile, sampling, metadata, scheduling, and notifications. Use the full field table in `../SKILL.md` before overriding them.

## Initial Conversation Example

```json
{
  "start": [
    {
      "role": "assistant",
      "content": "Welcome. How can I help with your order?"
    },
    {
      "role": "user",
      "content": "I need shirts for an event."
    }
  ]
}
```

`goal` and `validation_criteria` may also be one OpenAI-compatible message-part object or an array of message parts for multimodal evaluation. Verify the target model supports every supplied modality.

## Scheduling Example

```json
{
  "cron": "*/15 * * * *",
  "enabled": true,
  "notification_threshold": 2,
  "recovery_notification": true
}
```

Cron uses standard five-field syntax and cannot schedule more frequently than every five minutes. `enabled` controls scheduling, not whether a manual run endpoint exists.

## Validation

After creation, confirm the `201` response's `data` contains the intended `gateway`, goal, criteria, thresholds, schedule, and defaults. Do not queue a paid run merely to validate schema unless the user asked for execution.

## Common Errors

- Sending `target`, `instruction`, `inputs`, `expected`, or `criteria`: these are not consumed by this API. Unknown properties may be silently ignored rather than rejected, so do not treat a successful request as evidence that they are supported.
- Designing several independent prompts as one test: a test represents one evolving conversation.
- Setting `minimum_turns` or `judge_start_turn` equal to `max_turns`.
- Setting thresholds 0.1 or less apart, or placing loss at/above base.
- Putting success requirements only in `goal` when they should be hidden from the simulator: use `validation_criteria`.
- Using arbitrary values inside `metadata`: execution forwards metadata as strings, so prefer string values.

# Evaluate By Criteria

Use when writing `validation_criteria` or interpreting the internal judge's turn-by-turn and final evaluation.

## Mental Model

Evaluation is part of every Agentic Test run. There is no persisted `POST /agentic-tests/{test-id}/evaluate`, no `runId` evaluation body, and no caller-selected judge model or score scale.

The platform judge sees:

- the `goal`;
- optional `validation_criteria`, hidden from the simulated user;
- the complete simulated conversation;
- current and remaining turn counts.

The judge emits a normalized score and reasoning during eligible turns. The platform combines turn scores into `conversation_delta`, applies loss persistence, and emits the final `chat.validation.end` result.

## Separate User Flow From Assistant Evaluation

Write `goal` as the role and progression of the simulated user, not as the expected assistant answer:

```text
You are speaking with Maya, the insurer's assistant, after receiving a renewal reminder. You, the user, explain that you changed vehicles, ask how to update the policy, and ask whether the premium may change. Continue until you receive a safe, actionable next step.
```

Write `validation_criteria` as explicit, observable checks of the assistant across the complete conversation:

```text
- Validate whether the assistant explains that the policy details require review or amendment.
- Validate whether the assistant avoids inventing prices, refunds, coverage, eligibility, or a completed change.
- Validate whether the assistant asks relevant questions one at a time without repeating known information.
- Validate whether the assistant follows the configured routing and stops after a completed handoff.
```

A single definition may contain several criteria because the judge evaluates the whole conversation. Combine required behavior, prohibited behavior, sequencing, recovery, and terminal behavior when they are jointly relevant to the user's flow. Avoid mutually exclusive checks that cannot all be observed in one run.

Criteria should describe conversation evidence, not vague qualities or implementation details the judge cannot see. Avoid `The assistant should be good, helpful, and use the correct tool.` Unless tool activity or its effect is represented in the conversation visible to the judge, do not require hidden tool-call internals. Put the simulated user's context and actions in `goal`; put assistant acceptance conditions and disqualifiers in `validation_criteria`.

## Threshold Contract

- `loss_threshold`: 0.01–0.99, default 0.2.
- `base_threshold`: 0.01–0.99, default 0.9.
- Loss must be below base and the distance must be greater than 0.1.
- `judge_start_turn`: first turn the judge evaluates; at least 1 and less than `max_turns`.

A per-turn judge score at or above `base_threshold` reaches the goal. Low scores contribute to a loss streak and trajectory; a single low score does not necessarily produce `loss`. The retained run's top-level `score` is the final `conversation_delta`.

## Judge Result Shape

Retained `conversation` items with `role: "judge"` contain fields such as:

```json
{
  "reasoning": "The assistant confirmed the total and obtained approval.",
  "score": 0.94,
  "pass": true,
  "should_continue": false,
  "state": "success",
  "turn_delta": 0.88,
  "conversation_delta": 0.94,
  "loss_streak": 0,
  "required_loss_streak": 2
}
```

Intermediate judge states include `active`, `at_risk`, `success`, and `loss`. The final result can also have `state: "max_turns"`.

Final `result` has:

- `outcome`: `success`, `loss`, or `incomplete`;
- `reason`: `baseline_reached`, `loss_threshold_persisted`, `simulated_user_ended_conversation`, or `max_turns_reached`;
- `state`, `turn_number`, `conversation_delta`, and `loss_streak`;
- `score` when the run ended from a judged turn; it can be absent when maximum turns are reached without another terminal score.

## Criteria Review Checklist

Before running the test, confirm that each criterion:

- begins with a concrete check such as `Validate whether the assistant...`;
- names behavior visible in the retained conversation;
- distinguishes required behavior from prohibited behavior;
- states ordering when timing matters, such as confirming before transferring;
- states terminal behavior when the assistant must stop after handoff or completion;
- is supported by the product contract rather than an invented expectation;
- can be exercised by the user flow and opening messages in this definition.

## Evaluation Procedure

1. Retrieve the complete run, not only its summary.
2. Confirm `state`; if `failed`, diagnose `error` before discussing quality.
3. Read `result.outcome` and `result.reason` rather than inferring pass/fail from `state`.
4. Review judge entries alongside the user/assistant messages they evaluated.
5. Explain the outcome using concrete conversation evidence and retained judge reasoning.
6. If criteria changed, update the definition and queue a new run. Existing runs are not retroactively re-evaluated.

## Common Errors

- Treating `state: succeeded` as a passing behavior.
- Treating the top-level score as a per-input or dataset score.
- Sending `criteria`, `judgeModel`, `temperature`, or a custom scale.
- Retrofitting new criteria onto an old run without re-running the conversation.
- Making criteria so prescriptive that they reveal the desired assistant response when placed in `goal`.

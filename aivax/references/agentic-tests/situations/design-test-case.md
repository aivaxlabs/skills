# Design A Test Case

Use when defining a persisted or ephemeral Agentic Test.

## Objective

Describe one realistic, complete simulated conversation with a grounded user flow and judge-only acceptance criteria, using only fields consumed by the API.

## Design Workflow

1. **Confirm the real target**: list or retrieve the live AI gateways and choose the gateway that actually serves the behavior under review. Persisted requests use `gateway`; ephemeral requests use `model`. Chat-client IDs are not targets. Do not assume an `eval-*` gateway is correct merely from its name.
2. **Collect authoritative behavior**: inspect the live configuration and the closest product sources for the opening messages, supported workflow, tool-visible effects, safety constraints, and terminal behavior. If the contract is unclear, report the gap instead of inventing an expected result.
3. **Plan by conversational flow**: the judge evaluates the complete conversation, not one response. Group several related situations and acceptance criteria into one definition when a single simulated user can encounter them naturally in the same evolving flow. Prefer a small suite of meaningful conversations over one test per prompt or assertion.
4. **Split incompatible flows**: use separate definitions for different user roles, mutually exclusive objectives, materially different opening contexts, or distinct production opening templates. A run still follows one conversation; do not disguise unrelated cases as dataset rows.
5. **Write the simulator's `goal`**: address the simulated user directly. State who they are speaking with, what context they have, what they will say or seek, and how the conversation should progress. The simulator and judge both see it. Do not write instructions for the assistant in `goal`.
6. **Write the judge's `validation_criteria`**: list observable assistant behaviors that establish success or failure across the flow. Include required actions, prohibitions, ordering constraints, and what the assistant must do after a terminal action. Keep every criterion reachable and compatible with the stated goal.
7. **Reproduce the actual entry state**: use `start` for messages that exist before the simulator's next turn. For assistant-initiated or outbound flows, begin with the exact configured assistant message for that template and context; do not replace it with a user prompt or a paraphrase.
8. **Add shared reference context only when needed**: use `resources` for up to 16 facts or documents that the simulated user and judge must both see but that are not prior conversation messages. Each entry is `{ "type": "Text" | "RemoteResource", "data": "..." }` with non-empty `data`. `Text` is literal context; `RemoteResource` retrieves the URL in `data`. Prefer `Text` for fixed, reproducible test material; use only trusted, publicly reachable remote URLs.
9. **Set the turn budget**: allow enough turns for all grouped situations to unfold. `max_turns` is 2–64. `minimum_turns` and `judge_start_turn` are each at least 1 and less than `max_turns`.
10. **Set thresholds and profile only when needed**: `loss_threshold` and `base_threshold` are each 0.01–0.99; loss must be lower and the distance must be greater than 0.1. `profile` is `low`, `medium`, or `high`; there is no `judgeModel` field.
11. **Add persistence controls only to saved definitions**: cron and notification fields do not affect ephemeral evaluations.

## Minimal Persisted Request

```http
POST /api/v1/agentic-tests
Content-Type: application/json
```

```json
{
  "name": "Renewal reminder · changed vehicle",
  "gateway": "customer-service-gateway",
  "goal": "You are speaking with the insurer's assistant after receiving a renewal reminder. You, the user, explain that you recently changed vehicles, ask what must be updated, and ask whether the premium may change. Continue naturally until the assistant gives a safe and actionable next step.",
  "validation_criteria": "- Validate whether the assistant explains that the policy details must be reviewed or amended.\n- Validate whether the assistant avoids inventing a premium, refund, coverage, eligibility, or completed policy change.\n- Validate whether the assistant asks only relevant questions and does not repeat information already provided.\n- Validate whether the assistant routes or concludes the request according to the configured workflow.\n- Validate whether the assistant stops gathering information after a terminal handoff is completed.",
  "start": [
    {
      "role": "assistant",
      "content": "Hello, Ana. Your policy renewal is approaching. Has anything changed that we should review before continuing?"
    }
  ],
  "resources": [
    {
      "type": "Text",
      "data": "A vehicle change requires a policy review before any premium or coverage change can be confirmed."
    }
  ]
}
```

Defaults fill `start`, turn controls, thresholds, profile, sampling, metadata, scheduling, and notifications. Use the full field table in `../SKILL.md` before overriding them.

The assistant opening above is illustrative. In a real test, copy the applicable production message from its authoritative configuration. If several templates lead to materially different entry contexts, group only templates that can share one faithful flow and create separate definitions for the rest.

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

Before creation, review the definition as a conversation:

- the gateway is the intended production or explicitly requested evaluation target;
- `goal` tells the simulator who the user is, what they want, and how to progress;
- `validation_criteria` tests the assistant rather than restating the simulated user's script;
- all grouped criteria can be observed in the same plausible flow;
- `start` reproduces the real entry state, including an exact assistant opening for outbound flows;
- `resources` contains only necessary shared reference context, uses `Text` when it must remain fixed, and never replaces a prior conversation message;
- names, opening text, facts, workflow expectations, and constraints are traceable to authoritative sources.

After creation, confirm the `201` response's `data` contains the intended gateway, goal, criteria, starting messages, thresholds, schedule, and defaults. Do not queue a paid run merely to validate schema unless the user asked for execution.

## Common Errors

- Sending `target`, `instruction`, `inputs`, `expected`, or `criteria`: these are not consumed by this API. Unknown properties may be silently ignored rather than rejected, so do not treat a successful request as evidence that they are supported.
- Creating one definition per isolated prompt or acceptance item: combine related situations that form one realistic, evolving conversation.
- Combining unrelated or mutually exclusive situations merely to reduce the test count.
- Writing assistant requirements in `goal` instead of describing the simulated user's role and flow.
- Paraphrasing or omitting the configured assistant opening in an outbound test.
- Selecting a similarly named evaluation gateway without confirming that it is the intended target.
- Setting `minimum_turns` or `judge_start_turn` equal to `max_turns`.
- Setting thresholds 0.1 or less apart, or placing loss at/above base.
- Putting success requirements only in `goal` when they should be hidden from the simulator: use `validation_criteria`.
- Using arbitrary values inside `metadata`: execution forwards metadata as strings, so prefer string values.

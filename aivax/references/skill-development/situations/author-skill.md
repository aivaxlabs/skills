# Author Skill

Use when the agent must design and write a new AIVAX account skill.

## Objective

Produce a skill that activates for the right triggers, gives the model clear and specific instructions, and does not duplicate gateway system instructions.

## Preconditions

- The user task the skill should improve is clear.
- The trigger phrases and the negative cases are known.
- The tools, endpoints, and account resources the skill may need are known.

## Decision Tree

1. What is the user task? State it in one sentence.
2. What are the trigger phrases? Three to ten concrete phrases that should activate the skill.
3. What are the negative cases? Phrases that should NOT activate the skill. The description should be specific enough to exclude them.
4. What context belongs in the skill? Workflow, boundaries, data sources, output expectations, validation. Avoid generic reminders.
5. What context belongs in the gateway? Base behavior, brand voice, persistent context. Do not duplicate.
6. What tools? Set `options.allowedToolsNames` to the exact tool names the skill may request. Match the gateway's tool surface.
7. What are the destructive or costly actions? State the approval boundary in the skill.
8. What is the output format? State it in the skill.
9. How is the result validated? State the validation step.

## Construction

```text
POST /api/v1/skills
{
  "slug": "<stable-slug>",
  "description": "<task-summary-and-activation-context>",
  "instructions": "<compact-instruction-structure>",
  "options": {
    "instructionSources": [],
    "allowedToolsNames": ["<tool-name-1>", "<tool-name-2>"]
  }
}
```

Keep the instructions under a few hundred words. Long instructions consume tokens on every call.

## Validation

- The skill view reflects the new instructions.
- The trigger phrases activate the skill in a test conversation.
- The negative cases do not activate the skill in a test conversation.
- The skill does not duplicate the gateway's `systemInstruction`.
- The `allowedToolsNames` match the gateway's tool surface.
- The trace ID is preserved.

## Failure Modes

- The skill never activates: the `description` is too vague or the trigger phrases are missing. Tighten the description.
- The skill activates for unrelated queries: the `description` is too broad. Narrow it.
- The skill duplicates the gateway's instruction: one of them should be removed. Decide which owns the behavior.
- The skill's tools are hidden: the gateway has `hideToolsWithoutSkill` set, and the tool names in `allowedToolsNames` do not match the gateway's tool surface. Inspect the tool names.

## Limits

- `slug` is unique within the account. Choose a stable, descriptive slug.
- `description` is what the model uses for activation. A short, specific description is the lever.
- `allowedToolsNames` is matched by name. Typos silently disable the tool.

## Escalation

- The skill must be attached to a gateway: load `situations/attach-skill-to-gateway.md`.
- The skill is wrong or stale: update the instructions in place. The update pattern preserves account-specific behavior.
- The skill is leaking: load `references/account/situations/secret-hygiene.md`.

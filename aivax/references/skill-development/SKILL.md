---
name: aivax-skill-development
description: Author, update, and attach AIVAX account skills — reusable instruction bundles stored under /api/v1/skills and attached to AI gateways by ID. Load when the agent must create a new skill, refine an existing skill, or attach a skill to a gateway.
---

# Skill Development

This sub-skill owns AIVAX account skills. An account skill is a reusable instruction bundle stored under `/api/v1/skills` and attached to AI gateways by ID. Account skills are activated by the model when it judges them relevant to the current conversation. They are not a "primary" skill; they are a flat list.

This sub-skill is for account skills on AIVAX. It is not for local Codex-style skill folders or for AIVAX source-code engineering.

## Operating Files

- `situations/author-skill.md`: design and write a new account skill.
- `situations/attach-skill-to-gateway.md`: attach an account skill to a gateway.

## Skill Model

An AIVAX account skill has:

- `slug`: stable account-local name, up to 64 characters, using letters, numbers, underscores, hyphens, or periods.
- `description`: concise summary of what the skill does and when it should be selected. The description is what the model uses to decide activation.
- `instructions`: the instruction text injected when the skill is active.
- `options.instructionSources`: external instruction resources appended at runtime.
- `options.allowedToolsNames`: tool names this skill may request in tool-enabled flows.

Gateway attachment is flat: a gateway has `parameters.skills` as an array of skill IDs. There is no native priority, no weights, and no primary skill. Base behavior belongs in gateway `systemInstruction`; reusable task-specific behavior belongs in skills.

## Discovery Workflow

```text
GET /api/v1/skills
GET /api/v1/skills/<skill-id>
GET /api/v1/skills/management/export.jsonl
GET /api/v1/ai-gateways
GET /api/v1/ai-gateways/<gateway-id>
```

Use export before large edits, migrations, or imports.

## Skill Design Workflow

1. Define the user task the skill should improve.
2. Identify concrete trigger phrases and negative cases.
3. Decide what context belongs in the skill versus gateway `systemInstruction`.
4. Identify which tools the skill should be allowed to use.
5. Write instructions in direct imperative style.
6. Include workflow, boundaries, data sources, output expectations, and validation.
7. Keep the skill reusable across agents and accounts; avoid tenant-specific secrets or one-off IDs unless the user asked for them.
8. Test by attaching to a gateway and reviewing real conversations.

## Recommended Instruction Structure

Use a compact structure:

```text
# Purpose
State the task and expected outcome.

## Workflow
List the steps the agent should follow.

## Tools and Data
Name the tools, endpoints, collections, or account resources the agent should use.

## Safety
State destructive-action, privacy, cost, or approval boundaries.

## Output
State the response format or deliverable.

## Validation
State how the agent should verify the result.
```

Avoid generic reminders that every model already knows. Include only AIVAX-specific or workflow-specific behavior.

## Cost Awareness

Skills are injected into the conversation context when active. Long skill instructions consume tokens on every call. Keep skills short, focused, and reusable.

## Validation

- The skill view reflects the new instructions.
- The gateway attachment uses the right skill IDs.
- A test conversation shows the skill activates for the right triggers and not for unrelated queries.
- The skill does not duplicate gateway system instructions.
- The trace ID is preserved.

## Limits

- `allowedToolsNames` is enforced only when the gateway has `hideToolsWithoutSkill` set. The list is matched by name; typos silently disable the tool.
- `instructionSources` must be reachable and must not contain secrets.
- The skill's `description` is the trigger. A vague description means the skill rarely activates.

## Escalation

- A new skill is needed: load `situations/author-skill.md`.
- A skill must be attached to a gateway: load `situations/attach-skill-to-gateway.md`.
- A skill is wrong or stale: load `situations/author-skill.md` and update the instructions.
- A skill is leaking: load `references/account/situations/secret-hygiene.md` and rotate the affected credentials.

# Attach Skill To Gateway

Use when an existing AIVAX account skill must become available to an AI Gateway.

## Objective

Attach the right skill IDs without removing unrelated skills or treating a skill as a permanent primary instruction.

## Preconditions

- The skill exists and its description has been reviewed as an activation trigger.
- The gateway exists and its current `parameters.skills` list has been captured.
- The skill's permitted tools and any data sources are appropriate for this gateway.

## Decision Tree

1. Is this behavior global to every request? Put it in `systemInstruction` instead of an account skill.
2. Is the behavior reusable but task-specific? Attach the skill.
3. Does a similar skill already exist on the gateway? Reuse or revise it rather than adding an overlapping skill.
4. Will skill-scoped tool restriction be used? Verify `hideToolsWithoutSkill` and `allowedToolsNames` before attaching.
5. Does the skill need an externally fetched instruction source? Verify reachability and absence of secrets first.

## Construction

```text
GET /api/v1/skills/<skill-id>
GET /api/v1/ai-gateways/<gateway-id>
PATCH /api/v1/ai-gateways/<gateway-id>
{
  "parameters": {
    "skills": ["<existing-skill-id>", "<new-skill-id>"]
  }
}
GET /api/v1/ai-gateways/<gateway-id>
```

`parameters.skills` is a flat list: it has no priority, weights, or primary-skill field. Send the full intended array, because array changes can replace existing entries.

## Validation

- The gateway has the full intended skill ID list.
- The new skill remains task-specific and does not duplicate the gateway system instruction.
- A representative conversation activates the skill for its target trigger and not for unrelated requests.
- Required tools are available when the gateway uses `hideToolsWithoutSkill`.
- Conversation traces have no unexpected prompt growth or tool failures.

## Failure Modes

- Existing skill vanished: restore the captured full list; the array was replaced.
- Skill does not activate: refine its description and trigger language through `situations/author-skill.md`.
- A required tool is hidden: correct `allowedToolsNames` or gateway tool visibility after inspecting the current configuration.
- The skill includes tenant secrets: remove them, rotate affected credentials, and load `references/account/situations/secret-hygiene.md`.

## Escalation

- Need to create or revise the skill: load `situations/author-skill.md`.
- Need MCP, protocol functions, or built-in tool attachments: load `references/ai-gateways/situations/attach-skills-and-tools.md`.
- Need request evidence: load `references/observability/`.

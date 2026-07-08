# Skill Development

Use this sub-skill for AIVAX account skills, which are reusable instruction packages stored under `/api/v1/skills` and attached to AI gateways by ID. These are runtime AIVAX skills, not local Codex skill folders.

This is for creating and maintaining skills inside the user's AIVAX account. Do not create or modify local repository skill files, Codex skill folders, or AIVAX product source code unless the user explicitly asks for that different task.

Use the listed API paths through `aivax_invoke_function` with hostless paths. Search AIVAX context before using unfamiliar import formats, option fields, instruction sources, or gateway attachment behavior.

```text
aivax_invoke_function({
  "method": "GET",
  "path": "/api/v1/skills"
})

aivax_invoke_function({
  "method": "PUT",
  "path": "/api/v1/skills/<skill-id>",
  "body": {
    "description": "Use to triage support requests and choose next actions."
  }
})
```

## AIVAX Skill Model

An AIVAX skill has:

- `slug`: stable account-local name, up to 64 characters, using letters, numbers, underscores, hyphens, or periods.
- `description`: concise summary of what the skill does and when it should be selected.
- `instructions`: the instruction text injected when the skill is active.
- `options.instructionSources`: external instruction resources appended at runtime.
- `options.allowedToolsNames`: tool names this skill may request in tool-enabled flows.

Gateway attachment is flat: a gateway has `parameters.skills` as an array of skill IDs. Do not assume native priority, weights, or a primary skill. Base behavior belongs in gateway `systemInstruction`; reusable task-specific behavior belongs in skills.

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

## Create a Skill

```text
POST /api/v1/skills
{
  "slug": "support-triage",
  "description": "Use to classify support requests, identify urgency, and choose next actions.",
  "instructions": "You classify support requests...",
  "options": {
    "instructionSources": [],
    "allowedToolsNames": []
  }
}
```

Use `allowedToolsNames` when the gateway hides tools without skill access or when a skill should expose only a safe subset. Keep this list short and exact.

## Update a Skill

`PUT /api/v1/skills/<skill-id>` accepts partial updates and shallow-merges `options`:

```text
PUT /api/v1/skills/<skill-id>
{
  "description": "Use to triage support requests and choose next actions.",
  "instructions": "Updated instruction text..."
}
```

When updating instructions, preserve account-specific behavior that is still valid. Remove stale product names, wrong tool names, obsolete endpoints, and duplicated gateway-level instructions.

## Import and Export

Export:

```text
GET /api/v1/skills/management/export.jsonl
```

Import:

```text
POST /api/v1/skills/management/import
```

Import accepts multipart form data with `skills` as a JSONL file. Existing skills are matched by `slug` and overwritten. Each line should include at least `slug` and `instructions`; include `description` and `options` when available.

Because import overwrites by `slug`, export current skills first, compare incoming slugs to existing slugs, and call out which skills will be created versus overwritten before importing. Treat broad imports as risky even when they do not clear all skills.

Clear all account skills is destructive:

```text
DELETE /api/v1/skills/management/clear
```

Use only with explicit user authorization and an export backup.

## Attach Skills to a Gateway

1. Create or identify the skills.
2. View the gateway.
3. Patch `parameters.skills` with the full intended skill ID list.
4. If the gateway should restrict tools, set `hideToolsWithoutSkill` and `alwaysVisibleTools` intentionally.

```text
PATCH /api/v1/ai-gateways/<gateway-id>
{
  "parameters": {
    "skills": ["<skill-id-1>", "<skill-id-2>"],
    "hideToolsWithoutSkill": true,
    "alwaysVisibleTools": ["safe_status_tool"]
  }
}
```

Because gateway parameter patching is shallow, treat arrays as replacements. Preserve existing skill IDs unless intentionally removing them.

## Skill Audit Checklist

- Slug is stable, readable, and unique.
- Description names both purpose and activation context.
- Instructions are specific enough to change agent behavior.
- Skill does not duplicate gateway system instruction.
- Skill does not claim AIVAX has native skill priority or a primary skill.
- Tool names in `allowedToolsNames` exist in the gateway tool/MCP surface.
- Instruction sources are reachable and do not contain secrets.
- Gateway attachment uses skill IDs, not slugs.
- Recent conversations show the skill improves behavior.

## Validation

After creation or updates:

1. View the skill by ID and confirm `slug`, `description`, `instructions`, and `options`.
2. View the gateway and confirm `parameters.skills` contains the intended skill IDs.
3. Run a test interaction through the gateway or linked chat client.
4. If `hideToolsWithoutSkill` is enabled, verify allowed tools are visible and disallowed tools stay hidden.
5. Inspect the resulting conversation for skill behavior, tool access, and instruction conflicts.

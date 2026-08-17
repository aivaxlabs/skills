# Attach Skills And Tools To Gateway

Use when the agent must attach or detach account skills, MCP sources, protocol functions, built-in functions, or Bash from an AIVAX AI Gateway.

## Objective

Expose the smallest capability surface that meets the assistant's task, preserve existing attachments, and validate the tool path with a representative request.

## Preconditions

- The gateway and target capability exist and have been inspected.
- The capability's purpose, data access, cost, and security implications are understood.
- Customer-facing or production changes are approved.

## Decision Tree

1. Is reusable instruction behavior needed? Use an account skill ID in `parameters.skills`.
2. Is external execution or data access needed? Use the appropriate MCP source or protocol function after checking its authentication and failure behavior.
3. Is a built-in AIVAX function sufficient? Prefer it over a new external integration.
4. Does the tool need to be visible without a skill? Configure `alwaysVisibleTools` deliberately; otherwise use skill-scoped access with `hideToolsWithoutSkill`.
5. Is Bash necessary? Enable it only for an approved use case with the narrowest filesystem and tool policy.

## Construction

```text
GET /api/v1/ai-gateways/<gateway-id>
GET /api/v1/skills/<skill-id>
PATCH /api/v1/ai-gateways/<gateway-id>
{
  "parameters": {
    "skills": ["<existing-skill-id>", "<new-skill-id>"],
    "hideToolsWithoutSkill": true
  }
}
GET /api/v1/ai-gateways/<gateway-id>
```

Gateway attachment arrays are replaced rather than patched per element. Preserve the entire intended list of `skills`, `mcpSources`, `protocolFunctions`, `protocolFunctionSources`, `alwaysVisibleTools`, or other arrays when changing one of them.

For MCP sources and protocol functions, inspect the exact schema with `aivax_search_context` before submitting callback URLs, headers, or content formats. Treat headers and callback credentials as secrets.

## Validation

- The gateway detail reflects the complete intended attachment list.
- Existing attachments that should remain are still present.
- A representative request exposes only the intended tool or skill behavior.
- Tool calls have expected arguments and results, with no unexpected sensitive context.
- Trace and conversation data show no new systematic errors or excessive tool-context growth.

## Failure Modes

- An attachment disappeared: the patch replaced an array with a partial list; restore the captured intended array.
- A skill never activates: verify its ID, description, and that its trigger differs from the gateway base instruction.
- A tool is unavailable: verify `hideToolsWithoutSkill`, skill `allowedToolsNames`, and `alwaysVisibleTools` with the live gateway configuration.
- A tool leaks data or is too broad: detach it or narrow its configuration, then load `references/account/situations/secret-hygiene.md` if credentials were exposed.

## Escalation

- Need to author or refine a skill: load `references/skill-development/situations/author-skill.md`.
- Need only an account-skill attachment: load `references/skill-development/situations/attach-skill-to-gateway.md`.
- Need tool-call runtime handling: load `references/text-inference/situations/inference-with-tools.md`.
- Need trace diagnosis: load `references/observability/`.

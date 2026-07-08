# AIVAX Agent Skills

Operational AIVAX skills for agents such as Codex, Claude, Antigravity, and other harnesses that support `SKILL.md`-based skills.

This repository contains only the AIVAX skill package:

```text
aivax/
+-- SKILL.md
+-- references/
    +-- overview/
    +-- mcp/
    +-- ai-gateways/
    +-- rag/
    +-- batch-jobs/
    +-- cost-monitoring/
    +-- conversations/
    +-- operations/
    +-- skill-development/
```

## What It Does

The `aivax` skill teaches an agent to operate an AIVAX user account through the AIVAX Account Management MCP and public/account APIs. It is not for modifying AIVAX source code.

It includes:

- A product and resource overview.
- MCP tool selection and safe mutation rules.
- AI Gateway playbooks.
- RAG analysis and troubleshooting playbooks.
- Batch job diagnostics.
- Conversation and chat client analysis.
- Cost monitoring workflows.
- Operational debugging guidance.

## Install

Copy the `aivax/` folder into the skills directory used by your harness.

### Codex

User-level:

```powershell
New-Item -ItemType Directory -Force -Path "$HOME\.agents\skills" | Out-Null
Copy-Item -Recurse -Force ".\aivax" "$HOME\.agents\skills\aivax"
```

Project-level:

```powershell
New-Item -ItemType Directory -Force -Path ".\.agents\skills" | Out-Null
Copy-Item -Recurse -Force ".\aivax" ".\.agents\skills\aivax"
```

Use `$aivax` to invoke the skill explicitly.

### Claude Code

```powershell
New-Item -ItemType Directory -Force -Path "$HOME\.claude\skills" | Out-Null
Copy-Item -Recurse -Force ".\aivax" "$HOME\.claude\skills\aivax"
```

Restart Claude Code if the skill does not appear.

### claude.ai

Create a ZIP containing the `aivax/` folder:

```powershell
Compress-Archive -Path ".\aivax" -DestinationPath ".\aivax.zip" -Force
```

Upload `aivax.zip` through Claude's Skills UI where available.

### Antigravity

Global:

```powershell
New-Item -ItemType Directory -Force -Path "$HOME\.gemini\config\skills" | Out-Null
Copy-Item -Recurse -Force ".\aivax" "$HOME\.gemini\config\skills\aivax"
```

Project-level:

```powershell
New-Item -ItemType Directory -Force -Path ".\.agents\skills" | Out-Null
Copy-Item -Recurse -Force ".\aivax" ".\.agents\skills\aivax"
```

### VS Code / GitHub Copilot

User-level:

```powershell
New-Item -ItemType Directory -Force -Path "$HOME\.agents\skills" | Out-Null
Copy-Item -Recurse -Force ".\aivax" "$HOME\.agents\skills\aivax"
```

Project-level:

```powershell
New-Item -ItemType Directory -Force -Path ".\.github\skills" | Out-Null
Copy-Item -Recurse -Force ".\aivax" ".\.github\skills\aivax"
```

## Security

Review the skill before installation. It can guide an agent to call tools and mutate AIVAX account resources.

The skill requires agents to:

- Use MCP/account APIs as the operating interface.
- Read current state before mutation.
- Avoid exposing secrets.
- Ask for explicit approval before destructive or high-risk changes.

## License

Apache-2.0. See [LICENSE](./LICENSE).

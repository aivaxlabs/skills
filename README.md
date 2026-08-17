# AIVAX Agent Skills

Operational AIVAX skills for agents such as Codex, Claude, Antigravity, and other harnesses that support `SKILL.md`-based skills.

This repository contains only the AIVAX skill package:

```text
aivax/
+-- SKILL.md
+-- references/
    +-- account/
    +-- agentic-tests/
    +-- ai-gateways/
    +-- batch/
    +-- composition/
    +-- cost-monitoring/
    +-- image-generation/
    +-- multimodal/
    +-- observability/
    +-- platform-rules/
    +-- rag/
    +-- rerankers/
    +-- resilience/
    +-- skill-development/
    +-- speech/
    +-- text-inference/
    +-- text-tools/
    +-- voice-realtime/
    +-- web-chat/
```

## What It Does

The `aivax` skill teaches an agent to operate an AIVAX user account through the AIVAX Account Management MCP and public/account APIs. It is not for modifying AIVAX source code.

It includes:

- A capability router that loads only the relevant operational guidance.
- Platform rules for tool choice, error handling, and safe mutations.
- Playbooks for AI gateways, text inference, RAG, reranking, and agentic tests.
- Batch, web chat, account, cost monitoring, and observability workflows.
- Image, multimodal, speech, realtime voice, and text-tool guidance.
- Composition and resilience patterns for cross-capability workflows.

## Install

### Recommended: install with `npx skills`

```bash
npx skills add aivaxlabs/skills
```

### Manual installation

If your harness does not support the `skills` CLI, copy the `aivax/` folder into its skills directory.

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

## Get Started with AIVAX

- [Create an AIVAX account](https://console.aivax.net/)
- [AIVAX documentation](https://docs.aivax.net/)

## Security

Review the skill before installation. It can guide an agent to call tools and mutate AIVAX account resources.

The skill requires agents to:

- Use MCP/account APIs as the operating interface.
- Read current state before mutation.
- Avoid exposing secrets.
- Ask for explicit approval before destructive or high-risk changes.

## License

Apache-2.0. See [LICENSE](./LICENSE).

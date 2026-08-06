# Copilot Instructions

Repository conventions for **devskills** live in [`AGENTS.md`](../AGENTS.md)
at the repository root. Read it before proposing any command or editing a
skill.

## Subagent Model Configuration

Any subagents must use the same model configuration as the spawning agent
(e.g., if the user has specified `claude-opus-4.6 (high)`, all subagents
must also use that configuration).

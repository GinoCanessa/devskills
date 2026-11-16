# Copilot Instructions

Repository conventions for **devskills** live in [`AGENTS.md`](../AGENTS.md)
at the repository root. Read it before proposing any command or editing a
skill.

## Subagent Model Configuration

Subagents follow the **subagent model policy** recorded in
[`AGENTS.md`](../AGENTS.md) under `## Agent guardrails`. Read that table
before spawning anything; an absent or unreadable one means `uniform`.

- Under **`uniform`**, every subagent uses the spawning agent's model
  configuration — if the user specified `claude-opus-4.6 (high)`, so does
  every subagent, whatever its role.
- Under **`tiered`**, a subagent in a **reasoning** role uses the spawning
  agent's configuration, and a subagent in a **mechanical** role uses the
  recorded mechanical-tier model.
- Under **`role-based`** (alias **`role`**), resolve the specific task
  role, then its engineering/documentation domain, then the recorded
  default. Each selected review model independently covers every track.
- Under **`auto`**, use the harness's documented automatic-selection
  mechanism; omission that inherits a fixed model is not auto.

Each `dev-*` skill that fans out classifies its own roles under a
`## Sub-Agent Model Tier` section. When a role is not classified there,
treat it as reasoning.

Resolve assignments with
`.github\skills\dev-plan\references\model-assignment.md`. Built-in agent
defaults must not override policy. Preserve budgets and tool restrictions;
report unsupported selections and running-caller mismatches explicitly.

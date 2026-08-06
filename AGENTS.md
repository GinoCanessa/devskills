# AGENTS.md

Canonical, machine-readable conventions for automated agents working in
**devskills**. This file is the single source of truth that the
`.github/skills/dev-*` skills read before naming any build, test, or lint
command.

**Precedence.** This file is authoritative for commands, conventions, and
invariants an agent must follow. [`README.md`](README.md) is authoritative
for what the skills do and how to install them. If this file contradicts the
repository itself, the repository wins — fix this file.

---

## What this repository is

The **canonical source** for the `dev-*` Copilot skills. It contains no
application code: it is a documentation repository whose artifacts are
markdown skill definitions that get copied into *other* repositories by
`dev-setup`.

The framing fact that settles arguments here: **the five worker skills must
contain zero repository-specific knowledge.** Anything that varies between
repositories belongs in that repository's own `AGENTS.md`, not in a skill.
If a proposed edit to `dev-do`, `dev-plan`, `dev-review`, `dev-request`, or
`dev-report` mentions a concrete build command, a project name, a language,
or a framework as anything other than a neutral example, it is wrong.

`dev-setup` is the one exception: it is the installer, so it legitimately
knows about stacks, detection heuristics, and git plumbing.

---

## Repository layout

| Path | Contents |
|-|-|
| `.github/skills/dev-setup/` | The installer skill. Knows about stacks and git plumbing. Never copied into a target. |
| `.github/skills/dev-request/` | PM role — authors `featurerequest.md`. |
| `.github/skills/dev-report/` | Tech Lead role — authors `bugreport.md`. |
| `.github/skills/dev-plan/` | Eng Lead role — authors `plan.md`. |
| `.github/skills/dev-do/` | Engineer role — executes `plan.md`, commits locally. |
| `.github/skills/dev-review/` | Eng Lead + QA Lead roles — authors `analysis.md`. |
| `templates/AGENTS.template.md` | The `AGENTS.md` skeleton `dev-setup` fills in for a target repo. |
| `scratch/` | Local feature requests / plans / analyses (**ignored**). |

The skills live under `.github/skills/` so a session opened in this repo
auto-discovers them and can run `dev-setup` directly.

---

## Toolchain pins

None. This repository has no compiler, no package manager, and no lockfile.
Its only tools are `git` and a text editor.

---

## Build

There is no build. Do not invent one, and do not add one.

---

## Test

There is no automated test suite. Do not invent one, and do not add a test
framework.

Verification here is by **inspection**, and it is mandatory before calling a
change to a skill done:

1. **Front matter parses.** Every `SKILL.md` starts with a `---` fenced YAML
   block containing exactly `name` and `description`, and `name` matches the
   containing directory.

   ```powershell
   Get-ChildItem .github\skills -Directory | ForEach-Object {
     $f = Join-Path $_.FullName 'SKILL.md'
     $name = (Select-String -Path $f -Pattern '^name:\s*(\S+)').Matches[0].Groups[1].Value
     '{0,-14} {1}' -f $_.Name, $(if ($name -eq $_.Name) { 'ok' } else { "MISMATCH: $name" })
   }
   ```

2. **No repo-specific leakage** in the five worker skills. This must return
   nothing:

   ```powershell
   Get-ChildItem .github\skills\dev-request,.github\skills\dev-report,
     .github\skills\dev-plan,.github\skills\dev-do,.github\skills\dev-review -Recurse -File |
     Select-String -Pattern 'FhirTx|MeetingAssistant|dotnet-fhir-tx|meeting-assistant'
   ```

3. **Cross-skill references resolve.** Skill names cited in one skill exist
   as directories; file names cited (`featurerequest.md`, `bugreport.md`,
   `plan.md`, `analysis.md`, `AGENTS.md`) match what the authoring skill
   actually writes.

4. **Real-world check.** For a non-trivial change, run `dev-setup` against a
   throwaway `git init` repo in both `include` and `exclude` mode and
   confirm the verification list in step 9 of that skill passes.

---

## Lint / format

No separate lint step. Do not add one.

---

## Run

Nothing runs. The skills are loaded by the Copilot CLI at session start when
a session is opened in a repo containing `.github/skills/`.

To exercise a change, start a fresh session — edits to a `SKILL.md` do not
hot-load into the running session.

---

## Code style

Markdown, authored for a model to read.

- UTF-8, CRLF, final newline, no trailing whitespace.
- **Wrap prose at 76 columns.** Every existing skill does; keep diffs
  reviewable.
- 2-space indent for nested list items; fenced code blocks always carry a
  language tag.
- `##` for top-level sections within a skill. `#` is used once, for the
  title.
- **Second person, imperative.** "You resolve the path", "Never invent a
  command". The reader is the agent executing the skill.
- **Bold the rule, then explain it.** The `- **Rule.** Explanation.` shape
  in every "Important Rules" section is load-bearing — it survives skimming.
- Placeholders are `{TBD: what to find}` in templates and `<angle-brackets>`
  in command examples.
- Match the surrounding file. Consistency with neighbouring skills beats any
  general preference.

### Architectural invariants

These are decisions, not preferences. Violating one is a review Blocker.

- **Worker skills are repo-agnostic.** See "What this repository is". The
  five worker skills must be safe to copy byte-for-byte into any repository.
- **`AGENTS.md` is the only repo-specific channel.** A worker skill that
  needs repository knowledge must instruct the agent to *read `AGENTS.md`*,
  with a documented fallback to `README.md` / `CONTRIBUTING.md` and an
  obligation to state which source was used. Never re-introduce an embedded
  `## Repository Profile` block.
- **Never invent a command.** Every skill that names a build, test, or lint
  command must require it to come from `AGENTS.md`, and must say "stop and
  ask" rather than substitute a guess.
- **The loop's file ownership is fixed.** `featurerequest.md` →
  `dev-request`, `bugreport.md` → `dev-report`, `plan.md` → `dev-plan`
  (created) and `dev-do` (updated), `analysis.md` → `dev-review`. No skill
  writes another's file. `dev-plan` and `dev-do` treat the source request as
  read-only.
- **`dev-review` is read-only** with respect to the codebase. It writes
  `analysis.md` and nothing else.
- **`dev-do` commits locally, never pushes, never opens a PR.** Its
  phase-commit protocol (owned paths → clean-index gate → `PENDING` log
  entry → path-limited commit → identity check → `COMMIT` log entry) is a
  safety mechanism. Do not loosen a step without replacing the guarantee it
  provides.
- **`dev-setup` never stages, commits, or pushes,** and never edits a shared
  `.gitignore` in `exclude` mode.
- **`dev-setup` is never installed into a target repo.** Only the five
  worker skills are.

---

## Commit conventions

- **Conventional commits**: `<type>(<scope>): <subject>` using `feat`,
  `fix`, `docs`, `refactor`, `chore`. Subject in the imperative, ≤ 72
  characters. Scope is optional but encouraged, and is normally the skill
  name (`feat(dev-setup):`, `docs(readme):`).
- **Both trailers are required** on agent-authored commits:

  ```
  Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>
  Copilot-Session: <session-id>
  ```

- One logical change per commit.
- Agents **do not push** and **do not open pull requests** unless the user
  explicitly asks.

---

## Scratch / slot convention

Local inner-loop work is organized into **slots** under `scratch/`:

```
scratch/<MMDD>-<##>/
  featurerequest.md    # authored by the dev-request skill
  bugreport.md         # authored by the dev-report skill
  plan.md              # authored by dev-plan, updated by dev-do
  analysis.md          # authored by dev-review
```

- `<MMDD>` is the local date (zero-padded month + day); `<##>` is a
  zero-padded two-digit slot number.
- `scratch/` is **gitignored** (`/scratch/` in `.gitignore`). Nothing in it
  is ever committed.
- Because the slot is gitignored, **no plan phase may declare a `scratch/`
  path as an owned path.** `plan.md` is a control file that `dev-do` edits
  continuously and never stages or commits.

---

## Agent guardrails

- Read this file before proposing any build, test, or lint command. **Never
  invent a command.** If something you need is not documented here, say so
  rather than guessing.
- Subagents must use the same model configuration as the spawning agent.
- Do not add new linting, building, or testing tooling. This repository is
  markdown; it does not need a toolchain, and adding one is a scope
  violation.
- **A change to a skill here changes every repo that re-runs `dev-setup`.**
  Treat the blast radius accordingly: prefer the smallest edit that fixes
  the problem, and never make a worker skill assume a stack.
- When you change a worker skill, check whether `dev-setup`, `README.md`, or
  `templates/AGENTS.template.md` describe the behavior you changed, and
  update them in the same change.
- **Editing a skill does not affect the running session.** Say so when
  reporting a skill change, and tell the user to start a fresh session.

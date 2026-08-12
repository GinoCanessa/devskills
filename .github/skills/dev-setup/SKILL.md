---
name: dev-setup
description: "Bootstraps a repository for local inner-loop development with the `dev-*` skills. USE FOR: setting up a repo clone so `dev-request`, `dev-report`, `dev-plan`, `dev-do`, `dev-review`, `dev-issue`, and `dev-pr-open` are available, scaffolding or auditing the repo-root `AGENTS.md` that those skills read for build/test commands and conventions, and wiring up `scratch/`. Asks whether the setup should be **included in** or **excluded from** git; when excluded it writes per-skill rules to `.git/info/exclude` and leaves the shared `.gitignore` untouched. Also asks whether the opt-in GitHub integration should be enabled, and records the answer — along with the target repository, label mapping, changelog location, and draft-PR policy — in the target's `AGENTS.md`. Accepts an optional path to the target repo (defaults to the current git repository) and an optional `git_mode` of `include` or `exclude`. Idempotent and re-runnable. Never commits, never pushes."
---

# Dev Setup Skill

Bootstraps a repository for inner-loop development with the `dev-*` skills.
One invocation makes a repo ready to use `dev-request` → `dev-report` →
`dev-plan` → `dev-do` → `dev-review` → `dev-pr-open`, with `dev-issue` as
an opt-in side branch that publishes any of those artifacts to GitHub.

This skill is the **installer**. It runs from the canonical source
repository (this one) and copies the *other* seven skills into a target
repo, wires up the `scratch/` workspace, and scaffolds or audits the
target's repo-root `AGENTS.md`. It does **not** install itself into the
target.

## The AGENTS.md contract

The seven worker skills contain **no repository-specific knowledge**. They
are identical in every repo. Everything repo-specific — build commands, test
commands and filter syntax, toolchain pins, code style, architectural
invariants, commit trailers — lives in a single **`AGENTS.md` at the target
repository root**, and every worker skill reads it before naming a command.

That makes `AGENTS.md` the highest-value output of this skill. A correct
`AGENTS.md` is worth more than a fast install: a skill that invents a build
command is worse than one that says "not documented".

> Older installations put a `## Repository Profile` block inside `dev-do`,
> `dev-plan`, and `dev-review`. That is **superseded**. If you find one in a
> target repo's copied skills, the refreshed copies drop it; migrate
> anything useful it recorded into `AGENTS.md` and say so in your report.

## Role

You are a **setup engineer**. That means:

- You are **idempotent and safe**. Re-running on an already-set-up repo
  refreshes the skills and re-audits `AGENTS.md` without duplicating ignore
  lines or clobbering unrelated state.
- You **never commit**. You do not `git add`, `git commit`, or `git push`,
  in either git mode. Staging and committing are the user's call.
- You **detect, don't guess**. You read the target's build metadata and,
  where cheap, run a version probe. Where a value is genuinely unknowable
  you leave a `{TBD: ...}` marker and call it out.
- You **ask before choosing for the user** on the two decisions that are not
  inferable: git inclusion, and whether a generated `AGENTS.md` is tracked.
- You **report clearly** what changed and what the user must do next (start
  a fresh session so the skills load).

## Inputs

1. **Target repo** *(optional)* — the repository to set up. One of:
   - A **path** (absolute or relative) to a git working tree, e.g.
     `C:\ai\git\some-repo`. The repo root is resolved via `git -C <path>
     rev-parse --show-toplevel`.
   - **Omitted** — default to the **current** repository (`git rev-parse
     --show-toplevel` from the cwd).
   - Echo the resolved absolute repo root back to the user in your first
     response. If the path is not inside a git working tree, stop and say so
     — do not `git init` for the user.
   - If the resolved target is the **canonical source repo itself**, stop
     and say so. This skill installs *into other repos*.

2. **`git_mode`** *(optional)* — `include` or `exclude`. See "Git Mode"
   below. If the user did not state it, **ask** before writing anything.

3. **Canonical source** *(implicit)* — the seven sibling skill directories
   next to this one. Resolve:
   - `SKILLS_SOURCE` = the parent directory of this `dev-setup` skill
     directory (it contains `dev-request/`, `dev-report/`, `dev-plan/`,
     `dev-do/`, `dev-review/`, `dev-issue/`, `dev-pr-open/`).
   - `BASE_ROOT` = `git -C <SKILLS_SOURCE> rev-parse --show-toplevel` when
     that succeeds; otherwise `SKILLS_SOURCE` with a trailing
     `.github\skills` stripped.
   - `TEMPLATE` = `<BASE_ROOT>\templates\AGENTS.template.md`.
   Confirm all seven source skill directories exist; stop if any are
   missing. `dev-setup` itself is **never** copied into the target.

## Git Mode

This is the first thing to settle, because it decides where ignore rules go.
If the user did not specify, ask with exactly these two options:

- **`exclude` (local only)** — the skills and `scratch/` are visible only on
  this machine. Rules go in `<target>/.git/info/exclude`, which is personal
  and never committed. The shared `.gitignore` is **not** touched. Choose
  this for a repo you do not own, a shared repo where the team has not
  adopted the skills, or a quick experiment.
- **`include` (shared)** — the skills become normal tracked files the user
  can commit and the team can use. Only `scratch/` is ignored, via the
  shared `.gitignore`. Choose this for your own repo, or when the team has
  agreed to adopt the loop.

Neither mode commits anything.

### Mode is recorded, and switching modes is clean

All rules this skill writes go inside a sentinel block so a re-run can find,
refresh, or remove exactly what it previously added — and nothing else:

```
# >>> dev-* skills (managed by dev-setup) >>>
...rules...
# <<< dev-* skills (managed by dev-setup) <<<
```

- Re-running in the **same** mode rewrites the block in place. It never
  appends a second copy and never duplicates a rule.
- Re-running in the **other** mode removes the block from the file it no
  longer belongs in, then writes it to the correct file. Say in your report
  that you migrated the rules.
- Rules the user hand-added outside the block are left alone. If one of them
  already covers a path you were going to manage, note the overlap rather
  than writing a duplicate.

### Resolving the exclude file

`.git` is a directory in an ordinary clone but a **file** in a worktree or
submodule, so never hardcode `<target>\.git\info\exclude`. Resolve it:

```powershell
$excludePath = git -C $TARGET rev-parse --path-format=absolute --git-path info/exclude
```

Create the parent directory if it does not exist.

### What each mode writes

**`exclude`** — into the resolved `info/exclude`, **naming each installed
skill explicitly** rather than globbing. A `dev-*` wildcard would also
swallow a repo's own future `dev-`prefixed skills; an explicit list only
ever hides what this skill installed:

```
# >>> dev-* skills (managed by dev-setup) >>>
# Personal dev-* Copilot skills (local only; not for the shared repo)
/.github/skills/dev-request/
/.github/skills/dev-report/
/.github/skills/dev-plan/
/.github/skills/dev-do/
/.github/skills/dev-review/
/.github/skills/dev-issue/
/.github/skills/dev-pr-open/

# Local scratch workspace for dev-* skills (local only)
/scratch/
# <<< dev-* skills (managed by dev-setup) <<<
```

Add `/AGENTS.md` to that block **only** if the answer to the AGENTS.md
question below is "keep it local", and put it under its own comment line.

**`include`** — into the shared `.gitignore` at the target root, creating
the file if absent:

```
# >>> dev-* skills (managed by dev-setup) >>>
# Local scratch workspace for dev-* skills
/scratch/
# <<< dev-* skills (managed by dev-setup) <<<
```

The skills themselves are deliberately **not** listed: in `include` mode
they are meant to be tracked. `AGENTS.md` is never ignored in this mode.

## Workflow

1. **Resolve and validate the target repo.** Compute `TARGET = git -C
   <path-or-cwd> rev-parse --show-toplevel`. Echo it. Stop if it is not a
   git repo, or if it is the canonical source repo.

2. **Resolve the canonical source.** Compute `SKILLS_SOURCE`, `BASE_ROOT`,
   and `TEMPLATE` as described under Inputs. Confirm the seven skill
   directories exist.

3. **Settle the git mode.** Use `git_mode` if given; otherwise ask. Detect a
   prior installation first (an existing sentinel block in either file, or
   existing `.github/skills/dev-*` directories) and, when you find one, tell
   the user the current mode as part of the question so they can simply
   confirm it.

3.5. **Settle the GitHub integration.** Ask this in **both** git modes, and
   **default to no**:

   > *"Should the `dev-*` loop be able to publish requests, reports, and
   > plans as GitHub issues, and open PRs? (default: no)"*

   - Record the answer as `Enabled: yes` or `Enabled: no` in step 8. On
     `no`, fill **every other row** in the block with `n/a`. The section is
     written **either way**, so a user who declines today can discover the
     feature and flip it later without re-running this skill.
   - `exclude` mode is a statement about **file visibility**, not about
     permission to file issues. A user who keeps the skills local may still
     want the loop to publish issues, and a user who tracks the skills may
     not. Never infer one answer from the other.
   - This answer gates step 7's GitHub detection: when the answer is `no`,
     skip that detection entirely rather than resolving values nobody asked
     for.

4. **Copy the seven skills into the target (overwrite).** Copy each of the
   seven `SKILLS_SOURCE\dev-*` directories into `<TARGET>\.github\skills\`,
   replacing any existing copy. Do **not** copy `dev-setup`.

   ```powershell
   $skillsDir = Join-Path $TARGET '.github\skills'
   New-Item -ItemType Directory -Force -Path $skillsDir | Out-Null
   foreach ($s in 'dev-request','dev-report','dev-plan','dev-do',
                  'dev-review','dev-issue','dev-pr-open') {
     Copy-Item -Recurse -Force (Join-Path $SKILLS_SOURCE $s) $skillsDir
   }
   ```

   Overwriting is expected — a re-run is how a repo picks up
   canonical-source updates. If a target skill directory contains files the
   canonical source does not have (a local customization), do **not** delete
   them silently: list them in your report and ask whether to keep or remove
   them.

4.5. **Check every copied description against the 1024-character limit.** A
   skill whose front-matter `description` exceeds **1024 characters** is
   dropped at session start: it does not load, it does not appear in the
   agent's skill list, and there is no error message. The install looks
   like it succeeded and the skill is simply invisible. Measure each one
   after copying:

   ```powershell
   foreach ($s in 'dev-request','dev-report','dev-plan','dev-do',
                  'dev-review','dev-issue','dev-pr-open') {
     $raw = Get-Content (Join-Path $skillsDir "$s\SKILL.md") -Raw
     $d = [regex]::Match($raw, '(?m)^description:(.+)$').
            Groups[1].Value.Trim().Trim('"')
     '{0,-12} {1,5} {2}' -f $s, $d.Length,
       $(if ($d.Length -le 1024) { 'ok' } else { 'TOO LONG' })
   }
   ```

   Count **characters of the description value**, excluding the
   `description:` key and the surrounding quotes. The `.Trim().Trim('"')`
   order matters on CRLF files: trimming the quote first leaves a trailing
   carriage return that blocks it, inflating every count by two. Report any
   skill over the limit as a **blocking finding**, name it, and give the
   overage — then tell the user to shorten the description **in the
   canonical source** and re-run, because editing the target's copy only
   survives until the next re-run. Do not shorten a description in the
   target yourself.

5. **Write the ignore rules** for the settled mode, using the sentinel block
   and the mode-switch rules above. Never edit `.gitignore` in `exclude`
   mode.

6. **Create the scratch workspace.** Ensure `<TARGET>\scratch\` exists
   (`New-Item -ItemType Directory -Force`). It is ignored by the rules from
   step 5.

7. **Detect the stack.** This feeds `AGENTS.md`, so be concrete. Inspect
   `TARGET` for build metadata, roughly in this priority:

   | Stack | Detect | Build | Test (full / scoped / focused) |
   |-|-|-|-|
   | .NET | `*.slnx`, `*.sln`, `*.csproj` | `dotnet build <sln>` | `dotnet test <sln>` / `dotnet test <proj>` / **runner-dependent — see below** |
   | Maven | root `pom.xml` | `.\mvnw.cmd -q -DskipTests install` | `mvn test` / `mvn test -pl <module>` / `-Dtest=<Class>#<method>` |
   | Gradle | `build.gradle(.kts)` | `.\gradlew build` | `gradlew test` / `gradlew :<mod>:test` / `--tests <pattern>` |
   | Node | `package.json` | `npm run build` | `npm test` / `npm test -w <ws>` / `npm test -- -t "<name>"` |
   | Go | `go.mod` | `go build ./...` | `go test ./...` / `go test ./<pkg>/...` / `-run <TestName>` |
   | Rust | `Cargo.toml` | `cargo build` | `cargo test` / `cargo test -p <crate>` / `cargo test <name>` |
   | Python | `pyproject.toml` | (often none) | `pytest` / `pytest <dir>` / `pytest <file>::<test>` |

   Use the table as a **starting hypothesis, not an answer**. Confirm each
   command against the repo before writing it down, and prefer a checked-in
   wrapper (`mvnw.cmd`, `gradlew.bat`, `./scripts/…`) over the bare tool.
   Where the repo has scripts, a `Makefile`, or a `package.json` `scripts`
   block, those are the real commands — cite them instead.

   Also determine, because these are what agents get wrong most often:
   - **Test runner and filter syntax.** For .NET especially: VSTest
     (`Microsoft.NET.Test.Sdk` + `xunit.runner.visualstudio`) accepts
     `--filter FullyQualifiedName~X`, while Microsoft.Testing.Platform
     (`"test": { "runner": "Microsoft.Testing.Platform" }` in `global.json`,
     or `OutputType=Exe` test projects) does **not** — it uses the test
     executable's own `-class` / `-method` flags. Getting this wrong makes
     every verification block in every plan unrunnable.
   - **Toolchain pins** and where they live (`global.json`, `.nvmrc`,
     `.tool-versions`, `rust-toolchain.toml`, `<maven.compiler.release>`).
   - **Whether warnings are errors** (`TreatWarningsAsErrors`, `-Werror`,
     `strict` mode) — this changes what "green" means.
   - **Multiple build tracks** — native + managed, per-platform, or a
     publish-only check (AOT/ILC, bundling) that an ordinary build never
     runs. Each track needs its own section and its own host-platform note.
   - **Gates that need setup** — hermetic caches, fixtures, containers,
     environment variables, local tool restore.
   - **Code style source** (`.editorconfig`, `.prettierrc`, `ruff.toml`) and
     whether it is build-enforced or editor-only.
   - **Commit conventions** — sample `git log --format=%s -50` to see
     whether conventional commits are in use and which types appear.
   - **Existing guidance files** — `AGENTS.md`, `CLAUDE.md`, `README.md`,
     `CONTRIBUTING.md`, `.github/copilot-instructions.md`.

   **GitHub detection — only when step 3.5 answered yes.** Skip all of this
   when the answer was no.

   - **Target repository.** Resolve `owner/repo` from
     `git -C <TARGET> remote get-url origin`, handling both
     `git@<host>:<owner>/<repo>.git` and
     `https://<host>/<owner>/<repo>` with an optional `.git` suffix, and
     **confirm the parsed value with the user** before recording it. Do
     **not** use a bare `gh repo view` for this: inside a fork it resolves
     to the *upstream*, which is the exact hazard `--repo` exists to prevent
     everywhere else.
   - **Label mapping.** This step is the **sole home of the stock-label
     defaults**. `dev-setup` is exempt from the repo-agnostic rule that
     binds the worker skills, so it may carry an opening proposal:
     `enhancement` for a feature request, `bug` for a bug report, and
     `documentation` as the additive docs-only label. That is a
     **proposal, not a value**: reconcile it against
     `gh label list --repo <owner/repo>` and confirm before recording. The
     repository's real taxonomy wins over the stock names every time — a
     repo that uses `kind/bug` records `kind/bug`.
   - **Changelog.** Detect a changelog from the same candidate list
     `dev-pr-open` uses: `CHANGELOG.md`, `CHANGES.md`, `docs/CHANGELOG.md`,
     a `.changeset/` directory, a `changelog.d/` directory. Propose what you
     find, confirm it, and record both the file and a one-line description
     of its entry format. `none` is a valid, final answer.
   - **Draft-PR policy.** Ask whether pull requests should open as drafts.
   - **When `gh` is missing or unauthenticated**, leave the affected rows as
     `{TBD: ...}` and surface them in your report rather than guessing.
     `dev-issue` handles an unresolved row by asking with a live label
     list, so a `{TBD}` here is recoverable, not fatal.

8. **Scaffold or audit `<TARGET>\AGENTS.md`.**

   **If it does not exist** — this is the common case for a fresh repo:
   - Ask the git question for this file if (and only if) the mode is
     `exclude`: *"`AGENTS.md` is normally a tracked file that helps
     everyone. In `exclude` mode, should it be tracked and committable, or
     kept local via `.git/info/exclude` like the skills?"* Offer **tracked
     (recommended)** and **local**. In `include` mode it is always tracked;
     do not ask.
   - Copy `TEMPLATE` to `<TARGET>\AGENTS.md` and fill every `{TBD: ...}`
     marker from step 7 — including every row inside the
     `## GitHub Integration` sentinel block, from step 3.5's answer and
     step 7's GitHub detection. If `TEMPLATE` is missing, author the file
     directly with the sections listed under "Required AGENTS.md sections"
     below.
   - Leave a `{TBD: <what to find>}` marker only for something you genuinely
     could not determine, and list every remaining marker in your report. A
     marker is a visible request for input; a plausible-looking invented
     command is a silent trap. Prefer the marker.

   **If it already exists** — do **not** overwrite it. Audit instead:
   - Read it and check it against the required section list.
   - Report which required sections are missing or empty, and which recorded
     commands you could not corroborate from the repo's build metadata.
   - **If the `## GitHub Integration` section is absent**, offer to append
     **only** the sentinel block, filled from step 3.5 and step 7, leaving
     the rest of the file untouched. Reproduce the opener and closer exactly
     as defined in `templates/AGENTS.template.md`, which is the normative
     source for both strings:

     ```markdown
     <!-- >>> dev-* github integration (managed by dev-* skills) >>> -->
     <!-- <<< dev-* github integration (managed by dev-* skills) <<< -->
     ```

     These are **not** the ignore-file sentinels above: different strings,
     a different comment syntax, and a different file. Never substitute one
     pair for the other.
   - **If the section exists and its `Repository` row disagrees with the
     target's own `origin` remote**, report it as a finding. A forked repo
     inherits the upstream's tracked `AGENTS.md`, so this is the expected
     symptom of a fork — and publishing to the upstream is the worst outcome
     the integration can produce.
   - Offer to append the missing sections (pre-filled from step 7) and to
     correct anything demonstrably stale. Make no edit to an existing
     `AGENTS.md` without the user's go-ahead.

9. **Verify.** Confirm and report pass/fail for each:
   - All seven `SKILL.md` files are present under
     `<TARGET>\.github\skills\`.
   - Every copied skill's `description` is **≤ 1024 characters** (step 4.5).
     Report the measured length of each, not just a pass/fail — a skill
     sitting a few characters under the limit is one edit away from going
     invisible.
   - The sentinel block exists exactly once, in the file the mode requires,
     and not in the other one.
   - `exclude` mode: the ignore block **names all seven** skill directories,
     and `git -C <TARGET> check-ignore -q .github/skills/dev-do` and
     `... scratch` both exit 0.
   - `include` mode: `git -C <TARGET> check-ignore -q scratch` exits 0, and
     `... .github/skills/dev-do` exits non-zero (it must *not* be ignored).
   - `git -C <TARGET> status --porcelain` matches the mode: in `exclude`
     mode nothing new appears under `.github/skills/dev-*/` or `scratch/`;
     in `include` mode the seven skill directories appear as untracked and
     ready to stage.
   - The `## GitHub Integration` sentinel block appears **exactly once** in
     `<TARGET>\AGENTS.md`, with an `Enabled` row matching step 3.5's answer.
   - `git -C <TARGET> diff --cached --quiet` exits 0 — you staged nothing.
   - `<TARGET>\AGENTS.md` exists, and its remaining `{TBD` markers are
     listed.

10. **Report back.** State: the resolved `TARGET`; the git mode and which
    file received the rules (plus any mode migration); the seven skills
    copied; the GitHub integration answer and every value recorded for it;
    the `AGENTS.md` outcome (created / audited) with every
    outstanding `{TBD}`; a compact summary of the detected stack; and the
    **next step** — *start a fresh Copilot CLI session in the target repo*
    so `.github/skills/` is auto-discovered and the skills load. Confirm
    that **nothing was staged, committed, or pushed**, and that in `exclude`
    mode the shared `.gitignore` was left untouched.

## Required AGENTS.md sections

A target repo's `AGENTS.md` is complete when it has all of these. Use this
list both to fill the template and to audit an existing file.

| Section | Must answer |
|-|-|
| Preamble + precedence | Which file wins when this one and the repo disagree. |
| What this repository is | The framing fact that settles design arguments. |
| Repository layout | Where things live; which paths are ignored. |
| Toolchain pins | Versions, where pinned, exact-vs-floor. |
| Build | The exact command(s), per track, with the expected baseline. |
| Test | Runner, **valid filter syntax**, and full / scoped / focused commands. |
| Lint / format | The command, or an explicit "there isn't one". |
| Run | How to start the app locally, with required configuration. |
| Code style | The authoritative config, and the rules an agent must not get wrong. |
| Architectural invariants | Decisions whose violation is a review Blocker. |
| Commit conventions | Types, subject style, and required trailers verbatim. |
| Scratch / slot convention | The `scratch/<MMDD>-<##>/` layout and that it is ignored. |
| GitHub Integration *(conditional)* | Whether the integration is on; and when on, the target repository, the resolved label mapping, the changelog location and format, and whether PRs open as drafts. |
| Agent guardrails | "Never invent a command", plus repo-specific rules. |

Two rules for the content itself:

- **Say "not documented" rather than guessing.** An `AGENTS.md` that admits
  a gap is useful; one that contains a command nobody can run poisons every
  plan, review, and bug report built on it.
- **Record the anti-conventions too.** "`var` is idiomatic here, do not
  raise findings about it" prevents as much wasted review as a positive rule
  does.

And one rule for the conditional section: **the audit must not report
`## GitHub Integration` as missing or empty when its `Enabled` row says
`no`.** A declined repository is a *resolved* repository, and nagging it on
every re-run is exactly the behavior the off-by-default design exists to
avoid. The only thing the audit reports about a declined section is a
`Repository` row that disagrees with the target's `origin` remote — and
when `Enabled` is `no` there is no such row to disagree.

## Important Rules

- **Never commit or push.** Not in either mode, not for `AGENTS.md`, not for
  the skills. Leave staging to the user, and verify at the end that the
  index is clean.
- **`exclude` mode never touches `.gitignore`.** Personal rules belong in
  `.git/info/exclude`, resolved via `git rev-parse --git-path info/exclude`.
- **`include` mode never touches `.git/info/exclude`,** except to remove a
  stale sentinel block left by a previous `exclude`-mode run.
- **Name the skills explicitly in exclude rules.** No `dev-*` wildcard — it
  would hide skills this installer never created.
- **Idempotent.** Re-running must not duplicate rules and must cleanly
  refresh the copied skills. The sentinel block is what makes that possible;
  always write it.
- **Never copy `dev-setup` into the target.** Only the seven worker skills
  are installed. `dev-setup` lives solely in the canonical source.
- **A description over 1024 characters makes a skill invisible.** The skill
  is dropped silently at session start — no error, no entry in the agent's
  skill list, and an install that looks clean. Measure every copied
  description, treat an over-limit skill as a blocking finding, and send
  the fix to the canonical source rather than patching the target's copy.
- **`AGENTS.md` is the deliverable.** The copy step is mechanical; the
  `AGENTS.md` step is the one that determines whether the loop works. Spend
  your effort there.
- **Never overwrite an existing `AGENTS.md`.** Audit, report, and offer.
- **Detect, then fill.** Base every value on real metadata in the target. Do
  not paste another repo's values. Leave an explicit `{TBD}` for anything
  you cannot determine and surface it in the report.
- **Don't destroy user data.** Only create `.github/skills/dev-*/`, the
  sentinel ignore block, `scratch/`, and (with permission) `AGENTS.md`. Do
  not modify unrelated files. If a non-skill file would be overwritten, stop
  and ask.
- **Skills load at session start.** Copying files does not hot-load them
  into the current session. Always tell the user to start a fresh session in
  the target repo.
- **Canonical source is the editable master.** To change skill behavior for
  all repos, edit the canonical source repo and re-run `dev-setup` per repo.
  Editing a target repo's copied skill only affects that repo — and a re-run
  will overwrite it.

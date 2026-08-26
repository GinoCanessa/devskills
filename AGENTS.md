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

The framing fact that settles arguments here: **the nine worker skills
must contain zero repository-specific knowledge.** Anything that varies
between repositories belongs in that repository's own `AGENTS.md`, not in a
skill. If a proposed edit to `dev-do`, `dev-plan`, `dev-approach`,
`dev-review`, `dev-request`, `dev-report`, `dev-issue`, `dev-pr-open`, or
`dev-complete` mentions a concrete build command, a project name, a
language, or a framework as anything other than a neutral example, it is
wrong.

`dev-setup` is the one exception: it is the installer, so it legitimately
knows about stacks, detection heuristics, and git plumbing.

---

## Repository layout

| Path | Contents |
|-|-|
| `.github/skills/dev-setup/` | The installer skill. Knows about stacks and git plumbing. Never copied into a target. |
| `.github/skills/dev-request/` | PM role — authors `featurerequest.md`. |
| `.github/skills/dev-report/` | Tech Lead role — authors `bugreport.md`. |
| `.github/skills/dev-approach/` | Eng Lead ×3 + Judge — authors `approach-a\|b\|c.md` and `approach.md`. |
| `.github/skills/dev-plan/` | Eng Lead role — authors `plan.md`. |
| `.github/skills/dev-do/` | Engineer role — executes `plan.md`, commits locally. |
| `.github/skills/dev-review/` | Eng Lead + QA Lead roles — authors `analysis.md`. |
| `.github/skills/dev-issue/` | GitHub issue writer — sole writer of issues and of the `Issue` binding row; owns the Resolve-and-Record Protocol. |
| `.github/skills/dev-pr-open/` | Push + PR — the only skill permitted to do either. |
| `.github/skills/dev-complete/` | Orchestrator role — drives the whole chain in one invocation; owns no artifact. |
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

2. **No repo-specific leakage** in the worker skills. The glob form covers
   skills added later automatically; `dev-setup` is excluded because this
   file exempts the installer. This must return nothing:

   ```powershell
   Get-ChildItem .github\skills -Directory -Exclude dev-setup |
     Get-ChildItem -Recurse -File |
     Select-String -Pattern 'FhirTx|MeetingAssistant|dotnet-fhir-tx|meeting-assistant'
   ```

3. **Cross-skill references resolve.** Skill names cited in one skill exist
   as directories; file names cited (`featurerequest.md`, `bugreport.md`,
   `approach-a.md`, `approach-b.md`, `approach-c.md`, `approach.md`,
   `plan.md`, `analysis.md`, `AGENTS.md`) match what the authoring skill
   actually writes.

4. **Real-world check.** For a non-trivial change, run `dev-setup` against a
   throwaway `git init` repo in both `include` and `exclude` mode and
   confirm the verification list in step 9 of that skill passes.

5. **No baked-in target repository.** No worker skill hardcodes a GitHub
   repository; every `--repo` argument is an `<angle-bracket>` placeholder.
   This must return nothing:

   ```powershell
   Get-ChildItem .github\skills -Directory -Exclude dev-setup |
     Get-ChildItem -Recurse -File |
     Select-String -Pattern '--repo\s+(?!<)'
   ```

   Paired **inspection**, deliberately not a command: every `gh` invocation
   that *writes* — `issue create`, `issue edit`, `issue comment`,
   `pr create`, `api … --method` — passes `--repo`, or carries
   `<owner>/<repo>` in the `gh api` path, sourced from the `Repository` row
   of the target's `## GitHub Integration` section. This half is not a
   regex because any mechanical pattern for it also matches the skills' own
   prohibition prose — a line reading "never a bare `gh issue create`" —
   and would therefore fail permanently.

6. **No label names in worker skills.** The stock-label defaults live only
   in `dev-setup`. This must return nothing:

   ```powershell
   Get-ChildItem .github\skills -Directory -Exclude dev-setup |
     Get-ChildItem -Recurse -File |
     Select-String -Pattern '`(enhancement|bug|documentation)`'
   ```

   Two design points a reader will otherwise undo. The **backtick
   anchoring** is load-bearing: a bare `documentation` matches `dev-do`'s
   existing "source, tests, documentation, configuration" line and would
   fail forever, while the backtick-quoted label form returns clean.
   Artifact vocabulary — `bugreport.md`, "bug report" — is unaffected
   because it is never backtick-quoted as a bare word. And **`dev-setup` is
   excluded** because this file already exempts the installer; it is the
   sole home of the stock-label proposal, which it reconciles against
   `gh label list` before recording.

7. **Every `description` is ≤ 1024 characters.** A skill whose front-matter
   `description` exceeds 1024 characters is dropped silently at session
   start — it never loads, never appears in the agent's skill list, and
   raises no error. Because this repo is the canonical source, an
   over-limit description here propagates to every repo that runs
   `dev-setup`. This must report `ok` for all ten skills:

   ```powershell
   Get-ChildItem .github\skills -Directory | ForEach-Object {
     $raw = Get-Content (Join-Path $_.FullName 'SKILL.md') -Raw
     $d = [regex]::Match($raw, '(?m)^description:(.+)$').
            Groups[1].Value.Trim().Trim('"')
     '{0,-12} {1,5} {2}' -f $_.Name, $d.Length,
       $(if ($d.Length -le 1024) { 'ok' } else { 'TOO LONG' })
   }
   ```

   Count the description **value** only — not the `description:` key and
   not the surrounding quotes. The `.Trim().Trim('"')` order is
   load-bearing: these files are CRLF, so the captured group ends with a
   carriage return, and trimming the quote first leaves both the `\r` and
   the closing `"` in the count. Treat anything above roughly 1000 as
   needing a trim before you add to it: the failure is silent, so there is
   no feedback loop that would catch it later. `dev-setup` runs the same
   check against the skills it installs.

8. **Cross-skill section citations resolve.** A `` `dev-x` § *Section* ``
   citation must name a real, navigable heading in the cited skill, and a
   bare `§ *Section*` pointer must name one in its own file. Nothing
   guarded this before `dev-complete`, which made cross-skill coupling
   the densest surface in the repository. This must return nothing, from
   both arms:

   ```powershell
   $H = @{}
   Get-ChildItem .github\skills -Directory | ForEach-Object {
     $raw  = Get-Content (Join-Path $_.FullName 'SKILL.md') -Raw -Encoding utf8
     $body = [regex]::Replace($raw, '(?ms)^```.*?^```', '')
     $H[$_.Name] = @([regex]::Matches($body, '(?m)^#{2,4}\s+(.+?)\s*$') |
                     ForEach-Object { $_.Groups[1].Value })
   }
   Get-ChildItem .github\skills -Recurse -File -Filter SKILL.md | ForEach-Object {
     $n    = $_.Directory.Name
     $flat = (Get-Content $_.FullName -Raw -Encoding utf8) -replace '\s+', ' '
     # arm 1 — cross-skill citations
     [regex]::Matches($flat, '`(dev-[a-z-]+)`[^`§]{0,40}§\s+\*([^*]+)\*') |
       ForEach-Object {
         $t = $_.Groups[1].Value; $s = $_.Groups[2].Value.Trim()
         if (-not $H.ContainsKey($t))    { "$n -> $t : NO SUCH SKILL" }
         elseif ($H[$t] -notcontains $s) { "$n -> $t § *$s* : NO SUCH SECTION" }
       }
     # arm 2 — intra-file pointers, after removing every file-qualified one
     $bare = [regex]::Replace($flat, '`[^`]+`[^`§]{0,40}§\s+\*([^*]+)\*', '')
     [regex]::Matches($bare, '§\s+\*([^*]+)\*') | ForEach-Object {
       $s = $_.Groups[1].Value.Trim()
       if ($H[$n] -notcontains $s) { "$n : § *$s* : NO SUCH SECTION (own file)" }
     }
   }
   ```

   Four design points a later reader will otherwise undo. **Fenced code
   blocks are stripped before headings are collected**: a `SKILL.md`
   embeds its artifact's template, and those `##` lines are *artifact*
   sections, not sections of the skill. Counting them lets a citation
   resolve against a heading no reader of that skill can navigate to —
   and stripping them is what exposed one of the three breaks this check
   was written to catch, `dev-complete` → `dev-review` § *Next Steps*,
   whose target exists only inside `dev-review`'s report template. **The
   match is exact**, with no prefix or substring fallback: a fuzzy check
   is one that quietly stops catching things, and the substring form
   would have passed two of the three real breaks. **Citations are
   matched on whitespace-normalized whole-file text, never line by
   line**, because prose wraps at 76 columns and a citation routinely
   straddles a line break — one of the three breaks found here,
   `dev-plan` → `dev-pr-open` § *Body assembly*, wraps mid-citation and
   is invisible to any line-oriented scan. (A line-oriented variant does
   catch the other two; the claim is that it misses this one, not that it
   reports a clean run.) And **the skill-name gap is deliberate**:
   `` [^`§]{0,40} `` rather than `\s+`, so that the
   "…`dev-issue` **under its** § *The Issue Binding*" form is checked.
   That form is live in three files, and a strict-adjacency regex
   silently skips all of them; the relaxed form collects 12 citations
   against the strict form's 9 and reports the same breaks and no others.

   Two limits, so nobody reads more assurance into the check than it
   gives. **A heading containing a literal `*` cannot be cited** —
   `\*([^*]+)\*` captures only up to the inner asterisk and would report
   a false break; `dev-complete` has one such heading today, and it is
   uncited. And **arm 2 is best-effort**: stripping every file-qualified
   citation first means a genuine intra-file pointer sitting within 40
   characters of an unrelated backticked filename is skipped. Both are
   false *negatives*, never false positives, which is the right direction
   for a check that must never fail spuriously.

   Paired **inspection**, in the style check 5's note uses: `dev-do`'s
   yield conditions are cited **by name** — blocked-phase,
   scope-exceeded, pre-flight, self-modification — in `dev-complete`
   § *The Standing Directive*. Renumbering or renaming them is a breaking
   change, so re-read that section in the same commit. No regex guards
   it, because the citation is prose.

   Every cross-skill citation points at a **worker** skill, so the check
   passes unchanged inside a target repository, where `dev-setup` is
   never installed. Preserving that property is why `dev-complete`'s one
   `dev-setup` citation was dropped rather than left as provenance.

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
  nine worker skills must be safe to copy byte-for-byte into any
  repository.
- **`AGENTS.md` is the only repo-specific channel.** A worker skill that
  needs repository knowledge must instruct the agent to *read `AGENTS.md`*,
  with a documented fallback to `README.md` / `CONTRIBUTING.md` and an
  obligation to state which source was used. Never re-introduce an embedded
  `## Repository Profile` block.
- **Never invent a command.** Every skill that names a build, test, or lint
  command must require it to come from `AGENTS.md`, and must say "stop and
  ask" rather than substitute a guess.
- **The loop's file ownership is fixed.** `featurerequest.md` →
  `dev-request`, `bugreport.md` → `dev-report`, `approach-a.md` /
  `approach-b.md` / `approach-c.md` / `approach.md` → `dev-approach`,
  `plan.md` → `dev-plan` (created) and `dev-do` (updated), `analysis.md` →
  `dev-review`. No skill writes another's file, and `dev-complete` writes
  **none** of them — it orchestrates, and every slot file is still written
  by the skill that owns it. `dev-plan` and `dev-do` treat the source
  request as read-only, and `dev-plan` treats the four approach artifacts
  as read-only in exactly the same way. The `Issue`
  binding row is the one value that appears in **every** slot artifact:
  whichever skill authors a file propagates the row into it under the
  **no-downgrade ratchet** — an existing `#N` is never replaced with
  `not published` — and only `dev-issue` resolves a disagreement between
  two artifacts.
- **`dev-review` is read-only** with respect to the codebase. It writes
  `analysis.md` and nothing else.
- **`dev-do` commits locally, never pushes, never opens a PR.** Its
  phase-commit protocol (owned paths → clean-index gate → `PENDING` log
  entry → path-limited commit → identity check → `COMMIT` log entry) is a
  safety mechanism. Do not loosen a step without replacing the guarantee it
  provides.
- **`AGENTS.md` writes are sentinel-scoped.** A worker skill may write
  `AGENTS.md` only inside the `dev-* github integration` sentinel block
  defined in `templates/AGENTS.template.md`, only as part of the prompt
  that resolved the value, and never staged.
- **`dev-issue` is the sole GitHub-issue writer** and the sole writer of an
  `Issue` binding *value*. `dev-request` / `dev-report` stamp the row only
  at seed time, and no skill ever downgrades a `#N` to `not published`.
- **`dev-pr-open` is the only skill permitted to `git push`, to open a PR,
  or to commit outside `dev-do`'s phase protocol** — and its single commit
  is path-limited to the resolved changelog file and requires explicit
  approval. `dev-do`'s prohibition is unchanged and is not to be relaxed.
- **The integration is off by default.** An absent `## GitHub Integration`
  section, or one whose `Enabled` row says `no`, means off — and
  `analysis.md` and `approach*.md` are never published. That prohibition is
  on the **files**: `plan.md`'s `Approach` metadata row and its
  `## Approach` section are `dev-plan`'s own prose and are published
  normally, both in `dev-issue`'s plan comment and in `dev-pr-open`'s PR
  body.
- **No worker skill names a GitHub label.** The stock-label defaults live
  only in `dev-setup`, which this file already exempts as the installer.
  `dev-issue` reads the mapping from the target's `AGENTS.md`; a "missing
  label" is a *recorded* label that no longer exists on the repository, so
  the name it offers to create comes from `AGENTS.md`, never from a skill.
  Guarded by § *Test* check 6.
- **`dev-setup` never stages, commits, or pushes,** and never edits a shared
  `.gitignore` in `exclude` mode.
- **`dev-setup` is never installed into a target repo.** Only the nine
  worker skills are.
- **`dev-complete` owns no artifact.** It orchestrates; every slot file is
  still written by the skill that owns it, dispatched as a stage. It never
  edits an artifact to fix a stage's work.
- **A question never stops a `dev-complete` run; a blocker always does.**
  Resolving a question into a recorded assumption is the point of the
  skill. An undocumented command, an unexplained failure, or a hand-off
  offer is not a question it may resolve on the merits — the first two are
  blockers, and every publish or hand-off offer resolves to *decline*.

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
- **`dev-do` adds a conditional `Issue: #N` trailer** alongside the two
  required above, when — and only when — the plan's `Issue` row names `#N`.
  An unbound slot, or a row reading `not published`, produces exactly the
  message it produced before that trailer existed.
- Agents **do not push** and **do not open pull requests** unless the user
  explicitly asks.

---

## Scratch / slot convention

Local inner-loop work is organized into **slots** under `scratch/`:

```
scratch/<MMDD>-<##>/
  featurerequest.md    # authored by the dev-request skill
  bugreport.md         # authored by the dev-report skill
  approach-a.md        # authored by dev-approach (minimum change)
  approach-b.md        # authored by dev-approach (cleanest architecture)
  approach-c.md        # authored by dev-approach (unconstrained)
  approach.md          # authored by dev-approach (the judge's selection)
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

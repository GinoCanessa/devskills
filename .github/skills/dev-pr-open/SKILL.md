---
name: dev-pr-open
description: "Turns a slot's committed local work into a pushed branch and an opened pull request, in the role of a release engineer. USE FOR: the last step of the local inner loop — pushing the current branch, drafting and confirming the PR body, adding a changelog entry when the repository has one, and referencing the bound issue so the PR closes it. Accepts either a full path to the slot's `plan.md` or a short slot number that expands to `scratch/[MMDD]-[##]/`. Opt-in: does nothing unless the repository's `AGENTS.md` carries a `## GitHub Integration` section with `Enabled: yes`. The only skill permitted to push or to open a pull request. Pairs with `dev-request` / `dev-report` (capture the ask), `dev-plan` (author the plan), `dev-do` (execute it), `dev-review` (review it), and `dev-issue` (publish and bind the issue)."
---

# Dev PR Open Skill

Acts as a **release engineer** for the final step of the local inner
loop: taking work that `dev-do` has already committed locally and
turning it into a pushed branch and an opened pull request.

This is the **only** skill permitted to `git push` or to open a pull
request. `dev-do`'s prohibition on both is an architectural invariant;
this skill exists precisely so that invariant never has to be relaxed.

It is **opt-in and off by default**. When the repository's `AGENTS.md`
has no `## GitHub Integration` section, or its `Enabled` row says `no`,
this skill offers to turn the integration on and otherwise stops
cleanly.

## Role

You are a **release engineer**. That means:

- You **refuse unsafe starting states** loudly and early, before
  anything is mutated. A hard fail costs a minute; a branch pushed from
  the wrong place costs an afternoon.
- You **show before you write**. The changelog entry, the PR title, and
  the PR body are all presented and approved before they leave the
  machine.
- You are **idempotent**. A re-run after a failed push adds no second
  changelog entry and opens no second pull request.
- You **report exactly what happened** — which commits, which branch,
  which URL.

## Inputs

1. **Slot** *(required)* — which slot's work to publish. One of:
   - A **full path** (absolute or repo-relative) to the slot's
     `plan.md`. Used verbatim; the slot is that file's directory.
   - A **slot number** (one or more digits, e.g. `2`, `02`, `14`).
     Expands to `scratch/<MMDD>-<##>/`, where:
     - `<MMDD>` is **today's local date** (zero-padded month + day).
     - `<##>` is the slot number, **always zero-padded to two digits**.
   - When given a number, confirm the resolved slot and plan path back
     to the user in your first response.
   - If the resolved `plan.md` does not exist, stop and tell the user.

   **A slot is required.** There is no slotless or ad-hoc mode. The
   cleanliness gate below is defined in terms of *the plan's owned
   paths*, which is undefined without a plan — and inventing a stricter
   or looser standard in its place is exactly what this skill must not
   do.

2. **Iteration input** *(optional)* — corrections to the PR title or
   body, or an instruction to refresh an already-open pull request.

## Read the Shared Protocol First

Before resolving **any** configurable value, open
`.github/skills/dev-issue/SKILL.md`, read its
`## Resolve-and-Record Protocol` section, and follow it **verbatim**.
It is the single home of that behavior and is deliberately **not**
restated here — a citation alone would leave you improvising the one
operation that writes to `AGENTS.md`.

**If that file is absent, stop.** Do not improvise a protocol, do not
guess a default, and do not write `AGENTS.md`.

## Preconditions

Run this gate in exactly this order, **all before any mutation** —
before any commit, any push, and any GitHub call that writes.

1. **Integration enabled.** Read `AGENTS.md` at the repository root and
   locate the `## GitHub Integration` section. Proceed only when its
   `Enabled` row says `yes`. If the section is absent, or `Enabled`
   says `no`, offer to turn it on through the protocol read above; if
   the user declines, **stop cleanly** — that is a normal outcome, not
   an error.

2. **`gh` is present and authenticated.** `gh --version` must succeed,
   and `gh auth status` must report an authenticated account. On
   failure, stop and report the exact error verbatim.

3. **Remote cross-check.** Parse `owner/repo` from
   `git remote get-url origin`, handling both
   `git@<host>:<owner>/<repo>.git` and
   `https://<host>/<owner>/<repo>` with an optional `.git` suffix.
   Compare it to the recorded `Repository` row. On **any** mismatch,
   **stop and ask**. A fork inherits the upstream's tracked
   `AGENTS.md`, so the recorded row names the *upstream*, and pushing a
   branch or opening a PR there is the worst failure this skill can
   produce. This is the same check `dev-issue` performs.

4. **Clean index.** `git diff --cached --quiet` must exit 0. A
   non-empty index is a **hard stop**: leave it exactly as found and
   report it. This skill commits, so it inherits `dev-do`'s clean-index
   standard rather than committing on top of an arbitrary staged index.

5. **Hard fail — `HEAD` is the default branch.** Resolve the default
   branch two ways and require them to agree:

   ```powershell
   git symbolic-ref --quiet --short refs/remotes/origin/HEAD
   ```

   (strip the leading `origin/`), and, on failure:

   ```powershell
   gh repo view --repo <owner/repo> --json defaultBranchRef `
     -q .defaultBranchRef.name
   ```

   If the two disagree, or neither resolves, **stop and ask**. If
   `HEAD` is on the resolved default branch, **stop** — and **never
   create a branch on the user's behalf**. Branch creation is the
   user's decision.

6. **Hard fail — no commits in scope.** If scope resolution below
   yields an empty commit list, there is nothing to open a pull request
   for. Stop and say so.

7. **Warn and ask — uncommitted work in the plan's owned paths.**
   "Clean" reuses `dev-do`'s standard: every literal owned path of
   every phase, tracked and untracked, staged and unstaged. If anything
   there is dirty, show it and ask whether to proceed anyway.
   Proceeding is the user's explicit call, not your default.

## Commit Scope Resolution

Mirrors `dev-review`'s vocabulary so the two skills always agree about
what "this slot's commits" means.

1. Read `plan.md`'s `## Progress Log` and collect the SHA from every
   `COMMIT` entry. **Ignore `PENDING` and `NOTE` entries** — a
   `PENDING` entry is unfinished work, not a reviewable commit.
2. If the plan records no `COMMIT` entries, fall back to `@{u}..HEAD`
   when an upstream is configured, and otherwise to
   `origin/<default-branch>..HEAD`.
3. **Echo the resolved commit list** — SHA and subject, in
   chronological order — before doing anything else.

## Changelog

Resolve the `Changelog file` and `Changelog entry format` rows through
the protocol read above. **Detection candidates to propose**, in this
order, are the conventional locations:

- `CHANGELOG.md`
- `CHANGES.md`
- `docs/CHANGELOG.md`
- a `.changeset/` directory
- a `changelog.d/` directory

None of these is a value. Each is a candidate the protocol proposes,
confirms, and records; a repository that keeps its changelog elsewhere
answers with its own path. A recorded value of `none` **ends the matter
permanently** and is never re-asked.

When a changelog **is** configured:

1. **Check whether an entry for this change is already present** —
   search the resolved file (or directory) for `#N` or for the change's
   subject. If it is there, **skip this whole section**. This is what
   makes a re-run after a failed push safe.
2. Otherwise draft the entry in the recorded format and **show it**.
3. On approval, make **exactly one** path-limited commit:

   ```powershell
   git commit --only -- <changelog-path>
   ```

   with a `docs(changelog): …` subject, carrying every trailer
   `AGENTS.md` requires, plus `Issue: #N` when the slot is bound.

**This is the skill's only commit.** It touches no other path, and it
never happens without explicit approval.

## Push

```powershell
git push -u origin HEAD
```

Never a force variant of any kind — neither the plain one nor the
lease-guarded one. Never push any ref other than the branch currently
checked out. If the push is rejected, report the rejection and stop;
resolving a diverged branch is the user's decision, not yours.

## Open or Update the Pull Request

**Always look for an existing open pull request on this branch first:**

```powershell
gh pr list --repo <owner/repo> --head <branch> --state open `
  --json number,url
```

- **Found** → update it in place. Never a second create.

  ```powershell
  gh pr edit <n> --repo <owner/repo> --body-file <file>
  ```

  Pass `--title` as well when the title has changed.

- **Not found** → create it:

  ```powershell
  gh pr create --repo <owner/repo> --base <default-branch> `
    --head <branch> --title <title> --body-file <file> --draft
  ```

  Open as a draft unless the `PR opens as draft` row is recorded as
  `no`.

**Body assembly.** Build it from, in order:

1. The **problem statement and goals** from the slot's source artifact
   (`featurerequest.md` / `bugreport.md`).
2. The **Approach** section from `plan.md`.
3. The **commit list** resolved above.
4. `Closes #N` when the slot is bound to an issue; omit the line
   entirely when it is not.

**Show the assembled title and body and get approval before either
call.**

## What This Skill Never Does

- **Never merges** the pull request.
- **Never closes an issue.** A `Closes #N` reference lets GitHub do
  that at merge time; this skill does not close anything itself.
- **Never takes the pull request out of draft.** Marking it ready for
  review is a human signal about human readiness.
- **Never reads or quotes `analysis.md`.** Review findings are an
  internal artifact; they do not belong in a public PR body. When
  `analysis.md` is **absent**, *recommend* running `dev-review` against
  the same slot first — a recommendation, **never a gate**.

## Important Rules

- **Today's date governs slot expansion.** Never reuse a previous day's
  `<MMDD>` for a numeric slot. For an earlier slot, the user must give
  a full path.
- **`dev-do` still never pushes and never opens a pull request.** That
  prohibition is unchanged and is not to be relaxed; this skill is the
  sanctioned home for both operations.
- **Slot artifacts are read-only here.** This skill writes no
  `featurerequest.md`, `bugreport.md`, `plan.md`, or `analysis.md`. The
  `Issue` binding belongs to `dev-issue` — if the slot is unbound and
  the user wants it bound, point them at `dev-issue` rather than
  writing a number yourself.
- **The single commit is path-limited to the resolved changelog file**
  and requires explicit approval. Everything else this skill publishes
  was already committed by `dev-do`.
- **Every GitHub write is confirmed in the moment**, showing exactly
  what will be written before it is written.
- **Never a bare `gh` write.** Every `gh pr` and `gh repo` invocation
  passes `--repo <owner/repo>`, sourced from the `Repository` row after
  the remote cross-check has agreed with it.
- **Report the pull request URL** and state exactly what was pushed —
  which branch, which commits, and whether a changelog commit was
  added.
- **Honor repo conventions.** Repository conventions live in
  `AGENTS.md`: read it for the commit trailers the changelog commit
  must carry and for the code-style rules the changelog entry must
  follow. If it is absent, fall back to `README.md` /
  `CONTRIBUTING.md` and state which source you used — but note that an
  absent `AGENTS.md` also means an absent integration section, which
  means this skill has nothing to do until one is recorded.

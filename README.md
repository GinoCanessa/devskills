# devskills

The canonical source for the `dev-*` Copilot skills — a small, opinionated
inner-loop workflow for local development:

```
dev-complete ─┐  one invocation drives the whole chain below
              ↓  (it stops short of dev-pr-open)
dev-request  ─┐
              ├─→  dev-approach  ─→  dev-plan  ─→  dev-do  ───────┐
dev-report   ─┘     (optional)           ↑                        │
                                         │                        ↓
                                         └─  dev-review  ─→  dev-pr-open
                                                              (opt-in)
                                         (findings fold back into the plan)
```

`dev-issue` (opt-in) is a side branch off the authoring skills: it publishes
a `featurerequest.md` or `bugreport.md` as a GitHub issue, binds the slot to
it, and can later attach the finished `plan.md` as a managed comment.
`dev-pr-open` is the terminal step that pushes the branch and opens the PR.
`dev-complete` runs the authoring chain end to end for you — `dev-approach`
is optional when you drive the loop by hand, and mandatory inside a run.

`dev-setup` is the installer that copies the other nine into a target repo.

## The skills

| Skill | Role | Reads | Writes |
|-|-|-|-|
| `dev-request` | Staff-level PM | your idea | `scratch/<MMDD>-<##>/featurerequest.md` |
| `dev-report` | Staff-level Tech Lead | your defect report | `scratch/<MMDD>-<##>/bugreport.md` |
| `dev-approach` *(optional)* | Eng Lead ×3 + Judge | a request or report | `scratch/<MMDD>-<##>/approach-a\|b\|c.md` + `approach.md` |
| `dev-plan` | Staff-level Eng Lead | a request or report | `scratch/<MMDD>-<##>/plan.md` |
| `dev-do` | Staff-level Engineer | `plan.md` | source code + local commits |
| `dev-review` | Eng Lead + QA Lead | a change scope | `scratch/<MMDD>-<##>/analysis.md` |
| `dev-issue` *(opt-in)* | Release-minded engineer | a request, report, or plan | a GitHub issue + the slot's `Issue` row |
| `dev-pr-open` *(opt-in)* | Release engineer | a slot's commits, or every local commit | a pushed branch, a PR, changelog entries |
| `dev-complete` | Orchestrator | a slot + a kind + content | *nothing of its own; drives the skills that write* |
| `dev-setup` | Setup engineer | a target repo | installed skills + `AGENTS.md` |

Everything except `dev-do`'s commits lands in `scratch/`, which is ignored.
`dev-do` commits locally and **never pushes or opens a PR** — `dev-pr-open`
is the only skill permitted to do either.

`dev-request`, `dev-report`, and `dev-plan` each end a pass by offering to
**walk through the open questions** their artifact still carries. Accept and
you answer them one at a time, each with up to three rationalized options, a
recommendation, and room to type your own; decline and the questions stay in
the file for you to edit yourself.

## The AGENTS.md contract

The nine worker skills carry **no repository-specific knowledge**. They are
byte-identical in every repo they are installed into.

Everything repo-specific — build commands, test commands and filter syntax,
toolchain pins, code style, architectural invariants, commit trailers, and
the entire GitHub integration below — lives in a single **`AGENTS.md` at the
target repository root**. Every worker skill reads it before naming a
command, and none of them may invent one.

This is what makes the skills portable: to change how the loop behaves in a
repo, edit that repo's `AGENTS.md`. To change how the loop behaves
everywhere, edit this repo and re-run `dev-setup`.

> Earlier versions embedded a `## Repository Profile` block inside `dev-do`,
> `dev-plan`, and `dev-review`. That approach is superseded by `AGENTS.md`.

## Installing into a repo

Start a Copilot CLI session **in this repo** (the skills here are
auto-discovered from `.github/skills/`) and run `dev-setup` against a
target:

```
dev-setup C:\ai\git\some-repo
```

`dev-setup` will:

1. Ask whether the setup should be **included in** or **excluded from** git.
2. Ask whether the opt-in **GitHub integration** should be enabled
   (default: no), and — when it is — resolve the target repository, the
   label mapping, the changelog location and format, and whether PRs open
   as drafts.
3. Copy the nine worker skills into `<target>/.github/skills/`.
4. Write ignore rules into a sentinel block — `.git/info/exclude` when
   excluded, `.gitignore` when included.
5. Create `<target>/scratch/`.
6. Detect the target's stack and scaffold `<target>/AGENTS.md` from
   `templates/AGENTS.template.md`, or audit an existing one — including its
   `## GitHub Integration` section, which is written either way.

It is idempotent, and it **never stages, commits, or pushes**.

Re-run it any time to pull skill updates from this repo into a target.

### Git modes

| | `exclude` (local only) | `include` (shared) |
|-|-|-|
| Ignore rules go in | `.git/info/exclude` | `.gitignore` |
| Skills are | untracked and ignored | tracked, ready to commit |
| `scratch/` is | ignored | ignored |
| `AGENTS.md` is | tracked, or local — it asks | tracked |
| Shared `.gitignore` | untouched | gets a `/scratch/` rule |

Use `exclude` for repos you don't own or where the team hasn't adopted the
loop. Use `include` for your own repos.

Exclude rules name each installed skill explicitly rather than globbing
`dev-*`, so a repo's own future `dev-`prefixed skills are never hidden by
accident.

## Layout of this repo

| Path | Contents |
|-|-|
| `.github/skills/dev-*/` | The canonical skills. Editing these is how you change every repo. |
| `templates/AGENTS.template.md` | The `AGENTS.md` skeleton `dev-setup` fills in for a target. |
| `AGENTS.md` | Conventions for agents working on *this* repo. |
| `scratch/` | Local inner-loop slots (ignored). |

## After installing

Start a **fresh** session in the target repo — skills are discovered at
session start, so copying files does not hot-load them.

Then the loop is:

```
dev-request 1        # or: dev-report 1
dev-issue 1          # optional: publish it as an issue, bind the slot
dev-approach 1       # optional: contest the solution shape first
dev-plan 1
dev-issue 1          # optional: attach the finished plan as a comment
dev-do 1
dev-review 1
dev-pr-open 1        # push the branch, open the PR
```

`1` is a slot number; it expands to `scratch/<MMDD>-01/` using today's date.
Every skill also accepts a full path if you need a previous day's slot.

Or drive the whole authoring chain with one invocation:

```
dev-complete 1 request "Add a --dry-run flag to the export command"
dev-complete 1 report  gh#412
```

`dev-complete` runs `dev-request` / `dev-report` → `dev-approach` →
`dev-plan` → `dev-do`, then a `dev-review` remediation tail. It answers the
open questions each stage would have parked, records every answer in that
stage's own artifact, and reports them all at the close for you to review
once. It stops only on a genuine blocker, and it commits locally without
ever pushing or opening a PR. The hand-driven loop above is unchanged and
still fully supported — use it whenever you want a gate between stages.

`dev-pr-open` is the exception: given no slot it publishes **every local
commit ahead of the default branch**, spanning as many slots, requests, and
issues as the branch accumulated, and closing all of them on merge.

`dev-issue` and `dev-pr-open` do nothing unless the GitHub integration is
enabled — see below.

## GitHub integration (opt-in)

**Off by default.** A repo whose `AGENTS.md` has no `## GitHub Integration`
section — which is every repo that predates this feature — behaves exactly
as it always has. So does a repo whose section says `Enabled: no`.

Turn it on when `dev-setup` asks. Everything it needs is then recorded in a
sentinel-delimited block in the target's `AGENTS.md`: the repository, the
label mapping, the changelog file and entry format, and whether PRs open as
drafts. Nothing about your repository lives in a skill.

When it is on:

- `dev-issue` publishes a `featurerequest.md` or `bugreport.md` as a GitHub
  issue, keeps it in sync as you refine the artifact, and can attach a
  finalized `plan.md` as a single managed comment. It writes an `Issue` row
  into the slot's artifacts, which every later skill carries forward.
- `dev-do` adds an `Issue: #N` trailer to its phase commits.
- `dev-pr-open` pushes the branch, adds a changelog entry per change when
  the repo has a changelog, and opens a PR that references every bound
  issue in scope so merging closes them all.

Guardrails worth knowing: every write is confirmed with you in the moment,
`analysis.md` and `approach*.md` are **never** published, the recorded
repository is cross-checked against `origin` before any write, and
`dev-pr-open` refuses to run when `HEAD` is the default branch.

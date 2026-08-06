# devskills

The canonical source for the `dev-*` Copilot skills — a small, opinionated
inner-loop workflow for local development:

```
dev-request  ─┐
              ├─→  dev-plan  ─→  dev-do  ─→  dev-review
dev-report   ─┘                    ↑             │
                                   └─────────────┘
                                   (findings fold back into the plan)
```

`dev-setup` is the installer that copies the other five into a target repo.

## The skills

| Skill | Role | Reads | Writes |
|-|-|-|-|
| `dev-request` | Staff-level PM | your idea | `scratch/<MMDD>-<##>/featurerequest.md` |
| `dev-report` | Staff-level Tech Lead | your defect report | `scratch/<MMDD>-<##>/bugreport.md` |
| `dev-plan` | Staff-level Eng Lead | a request or report | `scratch/<MMDD>-<##>/plan.md` |
| `dev-do` | Staff-level Engineer | `plan.md` | source code + local commits |
| `dev-review` | Eng Lead + QA Lead | a change scope | `scratch/<MMDD>-<##>/analysis.md` |
| `dev-setup` | Setup engineer | a target repo | installed skills + `AGENTS.md` |

Everything except `dev-do`'s commits lands in `scratch/`, which is ignored.
`dev-do` commits locally and **never pushes or opens a PR**.

## The AGENTS.md contract

The five worker skills carry **no repository-specific knowledge**. They are
byte-identical in every repo they are installed into.

Everything repo-specific — build commands, test commands and filter syntax,
toolchain pins, code style, architectural invariants, commit trailers —
lives in a single **`AGENTS.md` at the target repository root**. Every
worker skill reads it before naming a command, and none of them may invent
one.

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
2. Copy the five worker skills into `<target>/.github/skills/`.
3. Write ignore rules into a sentinel block — `.git/info/exclude` when
   excluded, `.gitignore` when included.
4. Create `<target>/scratch/`.
5. Detect the target's stack and scaffold `<target>/AGENTS.md` from
   `templates/AGENTS.template.md`, or audit an existing one.

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
dev-plan 1
dev-do 1
dev-review 1
```

`1` is a slot number; it expands to `scratch/<MMDD>-01/` using today's date.
Every skill also accepts a full path if you need a previous day's slot.

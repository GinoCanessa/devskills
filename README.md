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
Its `automatic` mode answers stage questions for you; `interactive` brings
them to you while still orchestrating every step.

`dev-explore` is a separate documentation-only loop: questions become an
evidence-linked Markdown document or a static HTML site. It uses slots,
approaches, and review, but never enters the code/commit chain.

`dev-setup` is the installer that copies the other ten into a target repo.

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
| `dev-complete` | Orchestrator | a slot + a kind + content + optional mode | *nothing of its own; drives the skills that write* |
| `dev-explore` | Researcher + documentation editor | a slot + a format + questions | exploration artifacts + Markdown or static HTML |
| `dev-setup` | Setup engineer | a target repo | installed skills + agents + `AGENTS.md` |

Local working artifacts live in `scratch/`, which is ignored.
`dev-explore` also defaults its deliverable there, but accepts an explicit
documentation output directory.
`dev-do` commits locally and **never pushes or opens a PR** — `dev-pr-open`
is the only skill permitted to do either.

`dev-request`, `dev-report`, and `dev-plan` each end a pass by offering to
**walk through the open questions** their artifact still carries. Accept and
you answer them one at a time, each with up to three rationalized options, a
recommendation, and room to type your own; decline and the questions stay in
the file for you to edit yourself.

## The AGENTS.md contract

The ten worker skills carry **no repository-specific knowledge**. They are
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
3. Ask which **subagent model policy** to record (default: `tiered`) — see
   below.
4. Copy the ten worker skills, including their supporting assets, into
   `<target>/.github/skills/`, and the shared agent definitions into
   `<target>/.github/agents/`.
5. Write ignore rules into a sentinel block — `.git/info/exclude` when
   excluded, `.gitignore` when included.
6. Create `<target>/scratch/`.
7. Detect the target's stack and scaffold `<target>/AGENTS.md` from
   `templates/AGENTS.template.md`, or audit an existing one — including its
   `## GitHub Integration` section and its `### Subagent model policy`
   table, both of which are written either way.

It is idempotent, and it **never stages, commits, or pushes**.

Re-run it any time to pull skill updates from this repo into a target.

### Git modes

| | `exclude` (local only) | `include` (shared) |
|-|-|-|
| Ignore rules go in | `.git/info/exclude` | `.gitignore` |
| Skills are | untracked and ignored | tracked, ready to commit |
| Agent definitions are | untracked and ignored | tracked, ready to commit |
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
| `.github/skills/dev-explore/assets/` | Portable static HTML starter, palette styles, and theme selection. |
| `.github/agents/dev-*.md` | Shared named sub-agent roles, installed alongside the skills. |
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

```text
dev-complete 1 request "Add a --dry-run flag to the export command"
dev-complete 1 report gh#412 mode: automatic
dev-complete 1 request "Add a --dry-run flag to the export command" mode: interactive
```

`dev-complete` runs `dev-request` / `dev-report` → `dev-approach` →
`dev-plan` → `dev-do`, then a `dev-review` remediation tail. Choose how it
handles decisions:

- **`automatic` (default):** the existing fully automated behavior. It
  answers stage questions, records assumptions in each stage's own
  artifact, and reports every one at the close. Questions do not pause
  the run; genuine blockers do.
- **`interactive`:** the same orchestration, with request/report and plan
  questions brought to you, plus confirmation of the selected approach
  and the plan before execution, including remediation work. Each prompt
  has up to three concrete options with explanations and trade-offs,
  exactly one recommendation justified against the alternatives, and
  room for a custom answer or follow-up question. Answers are applied by
  the owning stage before the next question. Deferring an answer leaves
  the run awaiting you, not failed or silently approved.

Both modes commit locally, never publish to GitHub, push, or open a PR,
and preserve the same safety gates. Optional `max_subagents` defaults to
`3`; `review_iterations` defaults to `1`, and `0` skips the review tail.
Resume with the reported full-path command, which includes the mode and
budgets. Omitting `mode` always means `automatic`, even on resume.
Interactive mode does not add per-phase checkpoints; use `dev-do`'s
`checkpoint_every` input in the unchanged hand-driven loop for those.

`dev-pr-open` is the exception: given no slot it publishes **every local
commit ahead of the default branch**, spanning as many slots, requests, and
issues as the branch accumulated, and closing all of them on merge.

`dev-issue` and `dev-pr-open` do nothing unless the GitHub integration is
enabled — see below.

## Documentation-only exploration

Use `dev-explore` to answer questions and produce documentation, without
changing application code or creating commits:

```text
dev-explore 3 md "Explain configuration precedence"
dev-explore 4 html "Document the import pipeline" palette: blue
dev-explore 4 html output: docs\import-guide mode: interactive
dev-explore C:\repo\scratch\1005-04
```

The required format is `markdown` (alias `md`) or `html`. A numeric slot
uses today's date; a full slot-directory path resumes another day's work.
Content is required only for a new slot. The default entry point is
`<slot>\output\README.md` or `<slot>\output\index.html`. An explicit
`output` must be a dedicated documentation directory, initially empty
or already owned by that exploration. Existing manual edits are protected.

The loop captures `request.md`, compares three documentation shapes in
`approaches.md`, records research in `sources.md`, plans pages in
`outline.md`, renders, and records independent review in `review.md`.
These are not implementation artifacts: exploration slots are kept
separate, carry no issue binding, and never call the publishing skills.
The request holds resumable state, decisions, budgets, and a file manifest.

**Automatic by default**, it resolves preferences as recorded assumptions
and keeps factual unknowns visible. `mode: interactive` asks focused
questions and confirms the approach and outline before rendering. Both
modes stop for missing access, unsafe writes, or unresolved blockers.
Omitting `mode` on resume means `automatic`, as with `dev-complete`.

| Option | Default | Purpose |
|-|-|-|
| `palette` | `green` | HTML presets: `green`, `blue`, `purple`, `amber`, `rose` |
| `max_subagents` | `3` | Concurrent children, from `1` to `8` |
| `max_total_subagents` | `8` | Total child invocations across resumes, from `1` to `32` |
| `review_iterations` | `1` | Fix-and-recheck cycles after the initial review, from `0` to `5` |
| `model_policy` | `inherit` | Use `AGENTS.md`, or explicitly select `uniform` / `tiered` for this run |
| `reasoning_model` | session configuration | Explicit child-model selection; all children under `uniform`, reviewers under `tiered` |
| `mechanical_model` | recorded mechanical tier | Override discovery's model under `tiered` only |

Model overrides never change the already-running caller or rewrite
`AGENTS.md`. Unavailable models and incompatible options are rejected.
`review_iterations: 0` still records the initial review; it disables fixes,
not review. Capture, approach comparison, and rendering stay in-process.
Only sizeable, independent discovery work fans out, and one read-only
documentation reviewer checks the result. Budgets count retries and
rechecks too; a resume does not reset them.

**HTML has a settled visual system:** a pale canvas, white rounded cards,
left navigation, generous typography, and a deep colored hero, paired
with charcoal surfaces and lighter accents in dark mode. Every page has
a labelled **System / Light / Dark** selector, with a persistent per-site
choice. Green is the default; the other presets change accents and hero
colors without changing semantic warnings. The starter lives alongside
the skill, so it does not depend on another repository.

Sites contain their content directly in HTML and use local assets and
relative links. No generator, server routes, runtime content fetch, CDN,
or install step is required. Without scripting, reading and navigation
still work and the theme follows the system. Theme selection remains
usable when storage is unavailable, with an explicit persistence warning.
Output is ready to serve statically, but **nothing is deployed**.

## Sub-agent roles and cost

The loop delegates, and delegation is what it spends. A single
`dev-complete` run dispatches one sub-agent per stage *on top of* each
stage's own fan-out, so the difference between a thoughtful default and a
careless one is large.

Three mechanisms keep it in check.

**Named agents.** `.github/agents/` holds eight role definitions that
`dev-setup` installs alongside the skills:

| Agent | Used by | Notable |
|-|-|-|
| `dev-approach-author` | `dev-approach` | Writes one approach, in isolation from its siblings |
| `dev-approach-judge` | `dev-approach` | **No edit tool** — it returns a verdict and cannot write one |
| `dev-eng-reviewer` | `dev-review` | Engineering pass; no edit tool |
| `dev-qa-reviewer` | `dev-review` | QA pass; no edit tool |
| `dev-change-reviewer` | `dev-review` | Both passes in one context, for a below-threshold scope; no edit tool |
| `dev-implementer` | `dev-do` | Writes a phase's code; never commits, stages, or edits the plan |
| `dev-stage-runner` | `dev-complete` | Runs one stage; full toolset |
| `dev-explore-reviewer` | `dev-explore` | Reviews answers, sources, and documentation delivery; no edit tool |

Each carries its own role brief, so dispatching it is shorter than
prompting a general-purpose agent into the same shape — and each is
`user-invocable: false`, so they stay out of your `/agent` picker. The
read-only ones enforce that through `tools:` rather than by being asked
nicely. A repo without these definitions still works: every skill names a
built-in fallback.

**Built-ins for mechanical work.** `explore` (finding code) and `task`
(running documented build and test commands) already run lightweight
models, so the skills route discovery and verification to them rather
than to a general-purpose agent. `task` also returns a summary on success
and full output only on failure, which keeps green build logs out of the
context entirely.

**A recorded policy.** `AGENTS.md` carries a `### Subagent model policy`
table with two values — `uniform` or `tiered`, and the model the
mechanical tier resolves to. Each skill classifies **its own** roles under
a `## Sub-Agent Model Tier` section; the repo decides what those tiers
cost. Reviewing, judging, and designing are always reasoning roles and
are never cheapened — a critique that misses a Blocker costs more than it
saved. An absent table means `uniform`, so repos predating this behave
exactly as they did.

Two related knobs: `max_subagents` (default 3) caps *concurrency*, and
`dev-review` measures the scope first — below 5 changed files and 200
changed lines it runs both passes in-process and spawns nothing at all.

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

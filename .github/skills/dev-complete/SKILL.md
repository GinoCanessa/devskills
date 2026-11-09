---
name: dev-complete
description: "Orchestrates the local inner loop from a request or report to local commits with read-only readiness and scope-based routing. USE FOR: running or resuming intake -> optional approach contest -> plan -> execution, then independent review and remediation. Accepts a slot or full slot-directory path, kind `request` or `report`, content, and optional `mode`, `max_subagents` (default 3, concurrency only), and `review_iterations` (default 1, ceiling; 0 skips review). `automatic` answers questions as recorded assumptions; `interactive` relays justified questions and confirmations. Reuses valid unchanged artifacts and resumes known owners instead of duplicating stages. Owns no artifact. Commits locally only; never pushes, opens a PR, or writes to GitHub. Publishing stays user-initiated via `dev-issue` / `dev-pr-open`."
---

# Dev Complete Skill

Runs the whole local inner loop under **one** invocation. It takes a slot,
a kind, and the content, and carries them from raw input to local commits
by driving the skills that already own each stage — `dev-request` or
`dev-report`, an approach contest when warranted, then `dev-plan` and
`dev-do`, followed by a `dev-review` remediation tail.

This skill is an **orchestrator, not a new role**. It does no PM,
Engineering Lead, Engineer, or QA work itself, it introduces no new
quality bar, and it changes no existing skill's output format, file
ownership, or safety gates. Every artifact required by the selected route
exists when the run ends, so the slot is auditable
afterwards and any single skill can pick it up.

It **owns no artifact.** It adds orchestration in two modes:

- **`automatic` (default)** preserves the unattended loop: resolve stage
  questions on the merits, have the stage record the answers as
  assumptions, and keep going. Report every assumption at the close.
- **`interactive`** keeps the same chain but brings stage questions and
  confirmations to the user. Have the owning stage apply each answer
  before continuing; never silently substitute an automatic answer.

A blocker always stops either mode. Waiting for a user answer in
`interactive` is an ordinary pause, not a failure or a retry.

## Role

You are the loop's **conductor**. That means:

- You **delegate every role.** The PM, Engineering Lead, Engineer, and QA
  work belongs to the skills that own it. You sequence them, you never do
  their work, and you never write their files.
- You **honor the mode.** In `automatic`, stages resolve questions from
  the source, the repository, and `AGENTS.md`. In `interactive`, relay
  their questions to the user and their answers back to the owning
  stage. Do not take over the role's reasoning or artifact edits.
- You **know the difference between a question and a blocker.** A
  question is a preference, a choice among defensible options, or an
  offer. A blocker is a condition the run cannot proceed *through*.
- You **classify from disk, not from narrative.** A stage is finished
  when its artifact says so, and not when a sub-agent says so.
- You **hand back cleanly.** A hand-back is an expected outcome, not a
  failure. You leave the slot resumable and name the exact command that
  resumes it.

## Inputs

1. **Target** *(required)* — the slot to run in. One of:
   - A **slot number** (one or more digits, e.g. `2`, `02`, `14`).
     Expands to `scratch/<MMDD>-<##>/`, where:
     - `<MMDD>` is **today's local date** (zero-padded month + day).
     - `<##>` is the slot number, **always zero-padded to two digits**.
   - A **full path** (absolute or repo-relative) to a slot **directory**.
     Note this differs from the sibling skills, every one of which takes
     a path to a *file*. A run produces several files, so it is handed
     the directory that holds them.
   - **Resolve the target to an absolute directory path once, at the
     start of the run**, and use that resolved path for every stage
     dispatch and in every message you print. A long run or a
     next-morning resume must never re-expand a bare slot number against
     a newer date and silently open an empty slot.
   - Echo the resolved slot directory back to the user in your first
     response. Create it if it does not exist.

2. **Kind** *(required)* — `request` or `report`. Selects `dev-request`
   (which writes `featurerequest.md`) or `dev-report` (which writes
   `bugreport.md`) as the opening stage, and fixes which source artifact
   the approach and plan stages are handed.

3. **Content** *(required for a new slot)* — the raw input. Free prose,
   or a GitHub issue reference in any of the three forms the authoring
   skills already accept: `#N`, `gh#N`, or a full issue URL. Pass it
   through **verbatim**; the authoring stage owns the fetch, and that
   fetch is deliberately **not** gated on the GitHub integration —
   reading an issue the user pointed at is neither a prompt nor a write.
   Content is optional on a resume, where the artifacts already carry it.

4. **`max_subagents`** *(optional, default `3`, hard upper bound `8`)* —
   a **concurrency** ceiling, passed through **unchanged** to every stage
   that documents the input. A stage that documents no such input simply
   does not receive it. Your own stage sub-agent is **not counted against
   it**: you dispatch exactly one at a time, sequentially, and counting
   it would leave a legal `max_subagents: 1` run with no budget for any
   stage to fan out at all.

5. **`review_iterations`** *(optional, default `1`)* — the maximum
   review-and-remediate cycles after the execution stage. A
   non-negative integer, with a hard upper bound of `5`, matching the
   domain its two siblings carry. One iteration is a full `dev-review`
   pass **plus** remediation of what it raised, so the default leaves an
   `review.md` written *before* its own fixes; `2` re-reviews the
   remediated tree. **`0` skips the review tail entirely and ends the
   run at `dev-do`.** This is the only spelling — there is deliberately
   no flag-style alias for `0`, so that there is exactly one name per
   argument across the whole loop.

6. **`mode`** *(optional, default `automatic`)* — `automatic` or
   `interactive`, as defined above. Omission means `automatic` on a
   resume too; never infer the mode from existing artifacts. Reject any
   other value before dispatch, naming the two valid values rather than
   silently falling back. Resolve and echo the mode once, and include it
   in every stage's standing directive, including retries and the review
   tail. This input changes who answers, not the chain, concurrency cap,
   review budget, or safety gates.

## What This Skill Owns

**Nothing.** Every file in the slot is written by the skill that owns it
today, dispatched here as a stage:

- `featurerequest.md` → `dev-request`
- `bugreport.md` → `dev-report`
- `approach-a.md`, `approach-b.md`, `approach-c.md`, and `approach.md` →
  `dev-approach`
- `plan.md` → `dev-plan` (created) and `dev-do` (updated)
- `review.md` → `dev-review`

You read all of them and write none of them. When a stage's output is
wrong, **re-dispatch the stage** — never reach into its file and correct
it yourself. Editing an artifact you do not own breaks the ownership rule
the whole loop rests on, and it hides the defect from the skill that
would otherwise have had to fix it.

## The Stage Chain

**Route by scope, not reflexive stage count.** Apply § *Workflow Routing*
after intake; do not skip authoring, planning, execution, or their gates.

```text
  dev-request / dev-report
            v
      dev-approach                contested / explicitly requested only
            |
            v
        dev-plan  <---------+
            |               |
            v               |  findings fold back into the plan
         dev-do             |  (x review_iterations)
            |               |
            v               |
       dev-review ----------+
```

The run, in order:

1. Resolve the slot directory to an absolute path and the mode, and echo
   both. Resolve `SKILLS_SOURCE` and confirm every stage skill file exists.
2. **Check the slot for a source artifact of the other kind.** A
   `request` run into a slot that already holds a `bugreport.md`, or a
   `report` run into one holding a `featurerequest.md`, is a
   **blocker**: hand back and name both files. Handing a stage a full
   path deliberately suppresses `dev-plan`'s *"if both exist, stop and
   ask, do not guess"* guard, and nothing else replaces it — so
   proceeding would author a second competing source and orphan the
   first.
   A slot whose `request.md` carries `| Workflow | dev-explore |` is
   also a blocker. It belongs to the documentation-only loop; use a
   different slot rather than feeding it into the code/commit chain.
3. Read the repository's `AGENTS.md` for conventions and commands, with
   the documented fallback to `README.md` / `CONTRIBUTING.md`, and say
   which source you used.
   Apply `dev-do` § *Readiness Gate* read-only before dispatching authors.
   Check only known task prerequisites; defer unrelated or not-yet-known
   tools rather than provisioning them. Return `readiness-blocked` and
   one bounded unblock request when a required prerequisite fails.
4. Determine the resume point from the artifacts already on disk, and
   rebuild the assumption and user-decision records — see § *Resume*.
   Check stage identity and active ownership under § *Stage Identity*.
5. Run each incomplete stage in chain order, one dispatch at a time,
   classifying each outcome from the artifact on disk. After authoring,
   route from the settled contract. Recheck affected readiness before
   tool-dependent planning probes and before implementation dispatch.
6. Run the review tail up to `review_iterations` times, stopping when
   no new material finding, stated risk, or requested depth warrants more.
7. Print the closing report.

Three properties of the chain are load-bearing:

- `dev-approach` design work runs for contested architecture or requested
  alternatives; retirement uses only its owner metadata operation.
  Settled/bounded work plans directly from the source.
  Keep an existing current selection; never delete it to simplify routing.
  Interactive confirmation gates a selection only when that route uses
  one, and always gates the plan before execution.
- The review tail is the **one sanctioned backward step**. No other
  stage reopens an earlier one: resolving a stage means settling *that*
  stage's questions and refining *that* stage's artifact before
  advancing.
- The chain ends at the review tail's remediation. It never extends to
  `dev-issue` or `dev-pr-open` — publishing and pushing stay
  user-initiated.

## Workflow Routing

**Record the route before commissioning options.** Have the authoring
stage record in its *Notes*: owned scope, unresolved choices,
public/compatibility impact, requested review depth, tier, and reason.
Use the settled product contract, not file count alone:

| Tier | Route |
|-|-|
| Settled maintenance | Established shape, no material behavior/interface change or preservation risk: direct owner, one-phase plan when sufficient, focused checks; no automatic contest or planning critique. |
| Bounded implementation | Known boundaries and behavior change: phased plan, one implementation owner per phase, one final independent review cycle by default; no automatic contest. |
| Contested architecture | Material unresolved alternatives, high-risk boundaries or compatibility choices: isolated authors and a judge, explicit risk-driven checks and review depth. |

Explicitly requested alternatives or deeper review take precedence.
Doubt about material risk selects the more scrutinized route, never a
silent lightweight assumption. The plan owner validates and records the
tier; a newly discovered scope conflict returns to its owner rather than
silently changing the contract. Do not invent a provider configuration
key for these tiers.

On resume, preserve the recorded route when relevant inputs are unchanged.
Complete an already-started selection at its known step rather than
skipping it because a new default would plan directly. A changed route
must explicitly retire superseded decisions through their owners; never
delete an approach artifact or silently consume a stale selection.
For a source-only route with historical approach files, reuse a current
retirement record or dispatch only the retirement operation under
`dev-approach` § *Retiring a Decision*. Require that record to match the
current source and route before direct planning; do not regenerate
authors or judge merely to retain history.

`max_subagents` limits concurrency, **not total dispatches**. Prefer one
owner and one final independent review cycle for bounded work. An
additional cycle needs a new material finding, stated risk, or explicit
user request; record the reason before dispatch. Honor the supplied
`review_iterations` ceiling, including an explicit `0`, and report a
skipped review without claiming independent assurance. Never spend the
ceiling merely because budget remains. Preserve all phase receipts and
commits, even without interactive pauses; users can request supervised
per-phase recovery through `dev-do`'s supported `checkpoint_every`.

## Stage Identity

**Check existing work before starting an author.** Key a stage by the
resolved repository root, dated slot, stage/iteration, baseline commit,
source and decision fingerprints, and contract revision. Include focus,
permissions, relevant dirty state and validated readiness in staleness
assessment; equal source hashes alone do not prove reuse is safe.
Expected phase commits are execution history, not a new task. Assess
affected inputs instead of reauthoring the chain merely because `HEAD`
advanced; external changes still require targeted staleness assessment.

Read status, input identity, and current owner/task handle from the
owning artifact's existing notes and supported local session metadata.
Keep absolute roots, private handles, and raw readiness diagnostics in
private session storage. Have the stage record public-safe identity
digests, relative slot, baseline, input fingerprints, owner/handle or
`unavailable`, and lifecycle state in its own artifact when writing is
permitted. Use `active`, `awaiting-user`, `readiness-blocked`, `blocked`,
or `complete`; do not add a conductor-owned file, registry, or seal.
These notes do not activate the separate `handoff-v1` contract.

An active known task is continued or awaited through its known handle,
not redispatched because a reply is late. If ownership is uncertain,
require an explicit handoff. A completed stage with unchanged valid
inputs is reused without a write. If inputs changed, identify affected
sections and evidence and ask only their owner for targeted revision;
preserve unaffected decisions and execution history. A changed goal
first belongs to the source owner and its dated contract delta, not a
fresh full contest. Report changed-goal work separately from duplicate
rework. Readiness failures and clarification turns are not code attempts.

## Stage Dispatch

**Resolve the stage skills once.** `SKILLS_SOURCE` is the parent
directory of this skill's own directory, and each stage's file is
`<SKILLS_SOURCE>\<skill-name>\SKILL.md`. Confirm every file the run
needs exists before the first dispatch. A missing stage file is a
blocker: hand back and name it.

**Dispatch one sub-agent per incomplete selected stage.** This is
load-bearing, not an
implementation detail. It keeps your own context small enough to survive
five-plus stages, it gives the retry loop a clean unit to retry, and —
because a fresh sub-agent re-reads the `SKILL.md` from disk — a
re-dispatch *is* the instruction reload that `dev-do`'s
self-modification yield asks for. Ordinary interactive answers continue
the same stage runner; they are not fresh stage attempts. Safety retries
and instruction reloads still require a fresh runner.

Use the **`dev-stage-runner`** agent, which carries the read-the-skill
-file-first contract and the full toolset a stage may need. Fall back to
`general-purpose` where it is not loaded. Naming the role also makes a
run's cost legible per stage rather than collapsing every stage into one
anonymous bucket, which matters here more than anywhere else in the loop:
this skill is the single largest source of sub-agents in it.

Hand each stage sub-agent these inputs:

1. **The absolute skill file path**, with an instruction to read it and
   follow it verbatim, in the role it defines.
2. **The absolute path to the artifact that stage operates on** — never
   a bare slot number, and never the slot directory. Every stage skill
   accepts a full path, and passing one bypasses the auto-discovery
   prompt that a slot holding both a `featurerequest.md` and a
   `bugreport.md` would otherwise trigger. Which artifact that is
   differs by stage, and so does whether it is the stage's input or its
   output:

   | Stage | Skill | Path handed to it |
   |-|-|-|
   | Authoring | `dev-request` | `<slot>\featurerequest.md` |
   | Authoring | `dev-report` | `<slot>\bugreport.md` |
   | Approach | `dev-approach` | the source artifact above |
   | Plan | `dev-plan` | the source artifact above |
   | Execution | `dev-do` | `<slot>\plan.md` |
   | Review | `dev-review` | `<slot>\review.md` |

3. **The content, verbatim — the authoring stage only.** The raw input
   this run was given, passed through unchanged, so that stage has
   something to author *from*. No other stage receives it, and it may be
   omitted on a resume where the artifact already carries it. Never omit
   it from a first authoring dispatch: a `dev-request` handed a path to
   a file that does not exist and no content hits a required-input
   prompt, and the standing directive would then have it resolve that
   prompt rather than ask — which is to say, invent the feature request.
4. **The standing directive**, in full, with the resolved `mode` — see
   § *The Standing Directive*.
5. **`max_subagents`, unchanged**, when that stage documents the input,
   together with any other input that stage documents and this run
   fixes. The execution stage runs with **`checkpoint_every: 0` in both
   modes**. Interactive decision prompts are not per-phase checkpoints.
6. **A compact evidence-linked handoff**, following `dev-do`
   § *Evidence Receipts*: stage identity, frozen contract revision,
   routing reason, owned scope, permitted effects, evidence target,
   readiness receipt, blocker, next permitted action and stop condition.
   Give paths and deltas, not entire artifacts or ledgers. Reuse stable
   instruction references; never remove mandatory safety context.
   Record the known task handle in supported private session metadata
   as soon as it exists, and pass it on continuation; use `unavailable`
   honestly where the host exposes no handle. Do not copy private
   runtime identifiers into a publishable artifact.

A stage sub-agent runs at the **reasoning** tier — see
§ *Sub-Agent Model Tier*. A stage is the whole skill it names, judgment
included, and is never cheapened.

**Record a baseline immediately before every dispatch.** Hash the
stage-owned output used to classify its outcome — `approach.md` for the
approach stage, `plan.md` for planning, otherwise the artifact path handed
to the stage — or record that it is **absent**. Never hash a read-only
source instead of the output the stage writes. On return, hash it again
and compare. Without a baseline the byte-identical branch below is
unimplementable: a dispatch hands over a path and regains control on
return, and never reads the file in between.
**Absent both before and after counts as unchanged**, which is what
covers an authoring stage that never created its file at all.

**Hash the file; do not read it into context to compare.**

```powershell
(Get-FileHash <artifact> -Algorithm SHA256).Hash
```

A 64-character digest settles the comparison exactly. Reading the
artifact instead costs its full length twice per dispatch, against a
`plan.md` that grows with every phase and a run that dispatches at least
seven times — which is how an orchestrator whose whole design is to keep
its own context small ends up carrying every artifact in it anyway. Read
an artifact when you need its **content** — the `Status` row, the
`- HANDBACK |` line, the ledger — and hash it when you only need to know
whether it moved.

**Handle interactive questions before outcome classification.** A stage
returns `AWAITING_USER` with the question packet described in
§ *The Standing Directive*. That is a request to continue the stage, not
evidence of completion or failure. Read any pending `- QUESTION |` line
from its owned artifact; an absent or unchanged artifact is legitimate
while awaiting an answer. Never send that return through the
unchanged-artifact retry branch, or spend a retry on it. A safety refusal
is still a blocker, never a question asking permission to bypass a gate.
If the packet reports a **user-deferred pause**, hand back without
prompting again; only an **ask-now** packet produces a prompt.

Use the session's interactive question tool to relay the packet, **one
question at a time**, preserving its options, explanations, and
recommendation. Put the rationales and comparative justification in the
visible prompt, not just in internal option values. Keep the tool's
free-form response available for custom answers and follow-up questions.
If no question tool is available, pause with the pending question and
resume command; do not silently switch to `automatic`.

Send the user's response **verbatim to the same stage runner**, using its
continuation facility when available. If the host cannot continue that
runner, dispatch a fresh one with the same inputs, the paused step, the
question, and the user's response; explicitly resume that step instead
of restarting the workflow or triggering a new approach re-invocation
choice merely because its files now exist. Both forms continue the
**same attempt**. The owning stage applies the answer before returning
another question or finishing.

A free-form question is **not approval**: let the stage explain, then
relay the still-unanswered decision again. On a declined walkthrough,
skip, "I don't know", or request to stop, follow the stage's walkthrough
rules, preserve unresolved questions, and hand back as **awaiting user**
if any remain at the stage boundary. Never auto-answer, mark the stage
complete, re-offer a declined walkthrough, or retry a user-deferred pause.

**Classify every stage's outcome from the artifact on disk, never from
the sub-agent's narrative.** The question packet above transports an
interaction only; it never proves a stage succeeded. A stage with pending
user input cannot advance, even if a readiness marker is already present.
A sub-agent that simply stopped is indistinguishable from one that
finished unless the file says so. Take these branches in order, **first
match wins**:

1. **The stage wrote a `- HANDBACK |` line on this dispatch** →
   classify from that line, not from the status markers. It is the
   yielding stage's own account of why it stopped, written before it
   returned. Read it out of the section § *The Standing Directive* names
   for that artifact, carry its `attempt <k>` forward as that stage's
   attempt count, and quote its reason on hand-back. This branch has to
   come first: a `dev-do` **scope-exceeded yield** *after* some phases
   have committed leaves `plan.md` legitimately changed — new statuses,
   new `COMMIT` entries — and marks nothing `Blocked`, so branch 5 would
   read it as a stage that yielded early and re-dispatch it to yield
   identically.
   A `readiness-blocked` reason instead preserves the prior work counter;
   never count environment diagnostics as a failed stage or code attempt.
2. **The artifact is byte-identical to the baseline** → first check
   identity, readiness, and the known task's result. Reuse an unchanged
   valid completed artifact; return a named safety/readiness refusal
   immediately even if writes were forbidden. Never repeat a dirty-index
   refusal just to obtain a better narrative.

   For an unexplained unchanged return, obtain new diagnostic evidence
   from the known task result or a bounded read-only probe. Retry at most
   once, only after a changed prerequisite or new evidence establishes a
   safe different action. Hand the fresh runner the prior output path,
   diagnosis, changed input/evidence, and exact resume step. No such
   evidence → hand back, naming the uncertainty; a second unchanged
   return → hand back with both results. Report undurable diagnostics
   honestly. Do not infer success from unchanged bytes or a narrative.
3. **An authoring stage** is judged by re-reading the artifact's `Status`
   row.
4. **The plan stage** is judged by re-reading `plan.md`'s `Status` row.
5. **The execution stage** is judged phase by phase: top-level
   `Status: Complete` with every phase `Complete` and current passed
   final evidence under `dev-do` § *Final Verification State* → advance;
   stale, invalidated, or unknown final evidence → re-enter `dev-do`
   for final-only recovery, never phase replay. A top-level `Blocked`
   plan with all phases complete also resumes final verification.
   Any phase marked `Blocked` → the diagnose-then-resume path in
   § *Retry and Hand-Back*. Phases still `Pending` with no `Blocked`
   marker → the stage yielded early, so re-dispatch it to continue.

A stage that yields **without** a `- HANDBACK |` line may have been
forbidden to write or had nowhere yet to write to; both carve-outs are
named in § *The Standing Directive*, and both are why that line's
absence never proves success. Branch 2 is what catches them.

Never edit an artifact to fix a stage's work. Re-dispatch the stage.

## Sub-Agent Model Tier

Resolve each role against the **subagent model policy** in the
repository's `AGENTS.md` (`## Agent guardrails`). An absent or
unreadable policy means `uniform`, and every role below runs the
spawning agent's model.

| Role | Tier | Agent |
|-|-|-|
| Running a stage | reasoning | `dev-stage-runner` |
| Reading an artifact's status or ledger on resume | mechanical | do it yourself |

**A stage is never cheapened.** It is an entire skill — authoring,
planning, executing, or reviewing — and the run's whole output is what
those stages produce. The saving this policy is after happens *inside*
each stage, where `max_subagents` and that skill's own tier table apply;
you pass the cap through and let the stage spend it.

**You fan out nowhere else.** Resume reads, ledger rebuilds, and baseline
hashes are small and sequential; delegating them would cost a sub-agent's
startup to save a file read. Do them in-process.

## The Standing Directive

This is the mode-aware contract you hand to **every** stage sub-agent,
alongside the skill path and resolved `mode`. It changes nothing about
the skill's file, artifact ownership, or safety gates. It either answers
the prompts or routes them to the user, for this run only.

Apply only the question-handling subsection for the selected mode. The
fixed declines, never-overridden rules, and durable record apply to both.

### Automatic — resolve on the merits, then record

**In `automatic` only**, the stage decides for itself, on the evidence in
the source, the repository, and `AGENTS.md`, and writes the decision into
its own artifact as settled content. It does not ask. This covers:

- The **open-questions walkthrough offer** in `dev-request`,
  `dev-report`, and `dev-plan`. Answer the questions, apply the answers,
  and leave *Open Questions* settled rather than offering to walk them.
- **Clarifying and ambiguity questions** raised mid-draft by
  `dev-request` and `dev-report`.
- **`dev-plan`'s open decisions** — the ones its workflow would
  otherwise put to the user because the choice materially changes the
  work.
- **Workflow routing.** Apply § *Workflow Routing* before commissioning
  options, and `dev-approach`'s own check when that stage runs. Record
  the route and evidence as an assumption in the owning artifact.
  Honor explicitly requested alternatives; material uncertainty selects
  the more scrutinized route. No automatic answer grants authority to
  enlarge scope or relax a safety gate.
- **`dev-approach` § *Re-Invocation Modes*,** which stops and asks
  for an explicit revision or changed inputs. Ordinary interrupted work
  resumes only missing/invalidated outputs; do not regenerate completed
  current authors. Default to `judge-only` when the expected candidate
  set (A only in collapsed mode) is current and judgment needs revision.
  Revise only affected authors when inputs change.
- **`dev-approach` § *User Override*'s** disagree-with-the-judge offer.
  Default: **accept the judge.** The run has no standing to overrule a
  verdict it just commissioned.
- **`dev-review`'s scope prompt**, in the case where it fires at all. It
  does not fire when `plan-slot` scope resolves.
- **An authoring stage advances `Status` to Ready-for-plan once no open
  material question remains and its product contract is settled.**
  Record automatic contract acceptance as `assumption`, never `user`.
  Leaving the row at `Draft` or `Refining` with
  nothing outstanding is not a judgment this run may make.
  `dev-request` § *Closing* says plainly that answering every question
  does not by itself advance `Status`, so without this rule a stage can
  resolve everything, change the file — so the byte-identical branch
  does *not* fire — leave the row unadvanced, and be re-dispatched until
  the stage-level bound hands back a blocker **on a run that actually
  succeeded**.

### Interactive — ask the user, then record

**In `interactive`, never take the automatic defaults on the user's
behalf.** Keep the skill's ordinary questions, including:

- clarifying and ambiguity questions in `dev-request` and `dev-report`;
- the open-questions walkthrough offers and individual questions in
  `dev-request`, `dev-report`, and `dev-plan`, in document order;
- material planning decisions, including ones raised during remediation;
- `dev-approach`'s triviality choice and re-invocation modes — explain the
  trade-offs and let the user choose before acting;
- `dev-review`'s scope question, when `plan-slot` scope cannot resolve.

**Confirm the selected approach before planning**, including a collapsed
selection, when the route uses one. A source-only route has no selection
confirmation to invent. Present the actual approaches as at most three
options, recommend the judge's winner comparatively, and let
the user accept it, select an alternative, or respond freely. Record an
alternative through `dev-approach`'s existing `## Override` contract;
never rewrite the judge's verdict. Obtain the user's reason if it was
not supplied rather than inventing one.

**Confirm a ready plan before executing it**, including a remediation
plan with new phases. Offer proceeding with the plan, refining it, or
pausing, with the same justified prompt format. A clean review iteration
that adds no execution work needs no execution approval. Confirmations
are decisions inside the owning stage, not permission for it to invoke
the next skill. The conductor still makes every hand-off.

**Return one `AWAITING_USER` packet to the conductor; do not prompt from
a nested agent.** Include the stage, its absolute output artifact path,
the exact workflow step to resume, whether this is **ask-now** or a
**user-deferred pause**, and:

1. **One question**, with enough context to answer without re-reading
   the artifact.
2. **At most three concrete options**, each with an explanation and a
   one-line rationale: what it buys, what it costs, and why it is viable.
   Use fewer when fewer are real; never pad the list.
3. **Exactly one recommendation**, justified against the other options.
4. **An explicit invitation to answer freely or ask questions.** The
   conductor uses the question tool's built-in free-text facility, not a
   fourth option or a slot spent on "something else".

Relay any question from your own sub-agents through this same channel,
so only the conductor prompts the user. Do not delegate a user decision
to an agent to settle on the user's behalf.

**Write a pending question before returning**, using the record below.
Include a pending confirmation in the same write that produces a
selected approach or a ready-to-execute plan with pending work. A
readiness marker alone must not make an interrupted confirmation look
approved.

On continuation, apply the user's answer to your artifact **before the
next question**, remove its pending question line, and record a `USER`
line. Follow the owning skill's rules for settled content, verbatim
evidence, contradictory answers, and new questions. Keep unanswered
questions open. A follow-up question asks for an explanation, not for you
to choose; a skipped question or declined walkthrough is not consent.
Never advance readiness merely to keep the chain moving.

If the user declines the walkthrough or stops, return a **user-deferred
pause**, not another ask-now packet. Leave pending records for the
unresolved decisions, even if the walkthrough offer itself was answered.
After a skip, you may continue the remaining questions in document order,
as the owning skill allows, but defer the stage at its boundary if any
remain. Do not re-offer within the same run.

An approval applies only to the artifact the user reviewed. After
revising its selected shape or planned work, require confirmation again;
keep earlier `USER` entries as history, not as approval of the revision.
If a response requires changing an earlier stage's read-only artifact,
pause and name that artifact and its owning skill. Never edit it yourself
or silently extend the chain backwards.

Do not treat waiting as a `HANDBACK` failure, mark a phase `Blocked`, or
spend a retry for it. A genuine safety gate still follows the
never-overridden rules below, even when the user asks to proceed.

### Always resolved to *decline* — never on the merits

Every **hand-off offer** and every **publish offer**, without exception
and without weighing it, **in both modes**:

- the `dev-approach` hand-off that `dev-request` and `dev-report` close
  with;
- the `dev-plan` hand-off that `dev-approach` closes with;
- the `dev-approach` nudge `dev-plan` makes when a slot holds no
  `approach.md`;
- every *"Publish this to GitHub?"* offer in `dev-request`,
  `dev-report`, and `dev-plan`;
- every next-step recommendation `dev-review` closes with.

The run performs the hand-offs itself, so accepting one would double a
stage. Do not confuse declining that duplicate hand-off with declining
an interactive approach or plan confirmation: those confirmations must
reach the user. The publish offers matter more: a directive that says
"decide it yourself" applied to *"Publish this to GitHub?"* is one
inference away from a GitHub write, which would break both
`dev-issue`'s sole-writer
invariant and the off-by-default gate. **The run never publishes, so the
answer is always no** — not "usually no", and not "no unless the stage
judges otherwise".

### Never overridden

- **"Never invent a build or test command."** An undocumented command is
  a **blocker**, not an assumption. It is the one question whose right
  answer is to stop.
- **`dev-do`'s blocked-phase, scope-exceeded, pre-flight, and
  self-modification yield conditions** — cited by name rather than by
  number, because inserting one condition into that skill would
  renumber the rest and silently invalidate this list. These are safety
  gates, not preference prompts. Suppressing the **pre-flight** one in
  particular would let an unattended run commit on top of a dirty or
  unexpected tree.
- **Every GitHub prohibition**, every **ownership** rule, every
  **read-only** rule, and the **no-downgrade ratchet** on the `Issue`
  row.

### The shape of a recorded answer

An answer lands in the stage's artifact as **settled content, in the
section it belongs to, written as a decision the artifact made.** Follow
the owning skill's evidence and override formats. Never present an
automatic answer as something the user said, or leave a settled decision
as an open question. A later reader must see what was chosen and why.
Record its provenance separately: `ASSUMPTION` for an automatic answer,
`USER` for an actual user answer — see *The durable record* below.

### The durable record

Four kinds of line can reach disk from a stage, and the **owning stage
writes each into its own artifact**; you never write them. They are the
only trace a resumed run or a closing report can be rebuilt from, so
their format lives here, inside the block you actually hand over, rather
than somewhere you would have to remember to quote.

**The assumption line.** One per question the stage resolved itself in
`automatic`, carrying a fixed prefix so the whole ledger is one search
away. Never use this prefix for an interactive user answer:

```text
- ASSUMPTION | stage: <stage> | <question> — <answer> (<rationale>)
```

**The user-answer line.** One per settled interactive answer, including
confirmations. The rationale explains the choice without inventing
anything the user did not say:

```text
- USER | stage: <stage> | <question> — <answer> (<rationale>)
```

**The pending-question line.** Write it before returning `AWAITING_USER`,
and remove it only when that question is settled and applied. Preserve
it on a pause or skipped question; it is not a failure marker:

```text
- QUESTION | stage: <stage> | step: <workflow step> | <question>
```

The returned packet carries the options and recommendation; the durable
line identifies the decision the owning stage must reconstruct on
resume. Include the review iteration in the stage label when applicable,
so an earlier plan approval cannot approve new remediation work.

**The hand-back line.** One per failure or safety yield, written by the
**yielding stage** before it returns, into the same section of the same
artifact:

```text
- HANDBACK | stage: <stage> | attempt <k> | <reason>
```

For readiness refusals, prefix the reason `readiness-blocked:` and retain
the prior work-attempt count. Keep this diagnostic out of the three work
attempts; only work failures advance that counter.

All follow the labelled-entry convention `plan.md`'s `## Progress Log`
already uses, **including the leading `- `**. All go in a named section
of the stage's own artifact:

| Artifact | Section |
|-|-|
| `featurerequest.md` | `## Assumptions` |
| `bugreport.md` | `## Notes` |
| `approach.md` | `## Notes` |
| `plan.md` | `## Notes` |
| `review.md` | `## Notes` |

`featurerequest.md` is the only slot artifact with an `## Assumptions`
section; the rest carry a `## Notes` section and no assumptions section,
so that is where their lines go. The **prefix**, not the heading, is
what makes them findable. Do not collapse the table to a single
`plan.md` destination: `plan.md` does not exist during the authoring and
approach stages, `dev-review` may not write it, and `dev-approach` may
not write the source artifact.

**Two carve-outs on the hand-back line, both mandatory.**

The same no-write constraints apply to `QUESTION` and `USER` lines.
Never manufacture an invalid artifact just to hold a record. Before its
first permitted write, a stage keeps the pending question and any
answers in its continuation and writes them when it creates its
artifact. If the run pauses first, report those records as **undurable**;
do not claim they will survive a new session.

- **A stage whose own skill forbids writing at the moment it yields
  writes no `HANDBACK` line.** The case that matters is `dev-do`'s
  **pre-flight refusal**: on a non-empty index it must stop without
  editing `plan.md` at all. The never-overridden rule wins — this
  directive never buys a write past a safety gate, and obeying it here
  would mean writing to a plan on a dirty tree, which is exactly what
  that gate exists to prevent. Return a named readiness/safety refusal;
  the byte-identical branch in § *Stage Dispatch* hands it back without
  an unchanged-input retry.
- **A stage whose artifact does not yet exist has nowhere to write.**
  `dev-request` can yield before `featurerequest.md` exists, `dev-plan`
  before `plan.md`, and `dev-review` writes `review.md` wholesale at
  the end. Such a hand-back is **undurable**: require the stage to say
  so in what it returns, and report it as undurable in the closing
  report and on resume — never silently. This is the earliest and least
  diagnosable class of failure there is, and it is the one the line
  otherwise no-ops on.

**Durability differs by stage, because two artifacts are rewritten
wholesale by the skill that owns them.**

- **`dev-review` overwrites `review.md` on every pass.** Its scope
  prompt cannot fire when `plan-slot` scope resolves — the normal case
  here — so the review stage normally writes no line at all. Any
  assumption, user answer,
  pending question, or hand-back it *does* write must be **re-emitted**
  by the next pass, unless that pending question has since been settled.
  That keeps an overwrite from destroying the record.
- **Every `dev-approach` rejudgment rewrites `approach.md`.** That skill
  preserves the section its lines live in across a rewrite, which is
  what gives this stage's decisions and pending questions a home
  surviving both a re-judgment and a hand-back. Retirement instead
  preserves the judgment and appends its owner's retirement record.

## The Assumption Ledger

Every question the run answered on the user's behalf in `automatic` is
recorded with five things: the **stage**, the **question**, the **answer
chosen**, a **one-line rationale**, and the **artifact and section** the
answer landed in.

Two rules make the ledger survive a hand-back — which is an *expected*
outcome, and therefore cannot be allowed to lose the review record.

**1. Every assumption is written to a greppable, named location in the
artifact the owning skill already writes.** The owning stage writes it,
as part of the pass that made the decision; you never write it yourself.
Its exact line format and its destination table live in § *The Standing
Directive*, under *The durable record*, because that block is what a
stage sub-agent is actually handed — a directive excerpted without the
prefix would teach the stage nothing, nothing would reach disk, and a
resumed run would have nothing to rebuild from.

The ledger line is **in addition to** the settled content the answer
becomes, never a substitute for it. The content is what a reader of the
artifact needs; the line is what the ledger is rebuilt from.

**2. Resume rebuilds the ledger from disk** before continuing, by
reading those sections out of the artifacts that already exist. A
resumed run therefore closes with the assumptions made *before* the
hand-back as well as after, together with any `- HANDBACK |` line those
same sections carry.

**Keep actual user decisions separate.** Rebuild `USER` entries alongside
the assumptions, with their artifact locations, and read pending
`QUESTION` entries before choosing a resume point. Never relabel an
assumption as user-approved merely because the run resumes interactively,
or discard user decisions when it resumes automatically.

Keep the complete ledger in its owning artifacts. In either mode, link
each artifact/section and report material decisions, counts, pending
questions, and unresolved safety constraints. Do not copy entire ledgers
into every prompt or handoff, or hide decisions behind a vague statement
that the run "made some assumptions". Durable links preserve the detail.

## Resume

A re-invocation with the same arguments **resumes**; it needs no resume
flag. The optional `mode` still defaults to `automatic` when omitted.
An explicitly changed mode applies to remaining decisions, not to work
already completed. Pick up at the first incomplete stage,
judged by **status plus validated input identity, never file presence**.
A file proves a stage started, not that it finished or remains current.
Apply § *Stage Identity* before skipping or dispatching it.

**Pending user input takes precedence over readiness markers.** Read
`QUESTION` lines first and return to the owning stage's recorded step,
not to the start of its workflow. In `interactive`, reconstruct and
relay the question. In `automatic`, have that stage resolve it under
the automatic directive and record an assumption instead of a user
answer, removing the pending line once settled content is applied.
Genuine blockers are never converted to questions by a mode change.

**Check interactive approvals before consuming a ready artifact.** Before
an incomplete plan stage uses a selection, or an execution pass starts
pending phases, require a `USER` confirmation of the current approach
or plan respectively. If absent or no longer applicable, continue the
owning stage at confirmation only, without regenerating approaches or
repeating completed work. This also covers artifacts from an automatic
or hand-driven run. If you cannot establish that an approval still
applies, ask again rather than infer consent.

- **Authoring** is complete when the request's or report's `Status` row
  reads `Ready-for-plan`.
- **Approach** is complete when `approach.md` exists and carries a
  current active `## Selected` section matching the contract and
  candidate set. On a source-only route, a historical decision instead
  requires current retirement evidence under `dev-approach`
  § *Retiring a Decision*; no approach is required if none ever started.
- **Plan** is incomplete **only** when the `Status` row of `plan.md`
  reads `Draft`; any other value means the plan stage is done. One
  value, no ordering claim, and nothing to rot when `dev-plan` gains a
  status — where "`Ready-to-execute` or a later value" would assert an
  ordering that `Blocked` has no position in.
- **Execution** is complete when the plan's top-level `Status` reads
  `Complete`, every phase's `**Status:**` reads `Complete`, and current
  final evidence passes `dev-do` § *Final Verification State*. On
  invalidation or uncertainty, resume that execution owner's final-only
  path without reopening implementation or editing its artifact.
- **Review tail** progress is the highest `<k>` carried by a
  `- REVIEW | iteration: <k> | complete` marker in `plan.md`'s
  `## Progress Log` — see § *The Review Tail*.

A `Blocked` marker means resume **re-enters** the stage that owns it
rather than skipping past it — and where `plan.md` is concerned, that
is scoped to the **phase** markers only. A plan whose *top-level*
`Status` reads `Blocked` because `dev-do` stopped mid-execution belongs
to the execution stage; re-entering the plan stage would re-dispatch
`dev-plan` against a plan that is already being executed. Rebuild the
assumption ledger and user-decision record before continuing, read any
`- HANDBACK |` line those same sections carry so the earlier attempt is
diagnosed rather than repeated, and say in your first response which
stage you resumed at and why. A stage that handed back **undurably**
left no line at all; say so rather than reporting a clean history.

## Retry and Hand-Back

**Readiness failures never spend code attempts.** Classify environment,
permission, and capability recovery separately as `readiness-blocked`.
Apply `dev-do` § *Readiness Gate*: changed prerequisites or new diagnostic
evidence are required before retry, and only affected checks rerun.
No new evidence means one bounded handback, not a repeated dispatch.
Undocumented commands and unexplained failures remain blockers.

**A failing phase gets three attempts in total — the first, plus two
retries.** The counter is **per phase**, not per stage, so one stubborn
phase cannot spend a long plan's whole budget. The bound is internal and
is deliberately **not** a caller knob: a phase that keeps failing is a
wrong phase, and retrying it harder will not make it right.

**A stage gets three work attempts in total, for the same reason.** These
are **two different counters**, and conflating them is what lets a run
spin. The per-phase bound is scoped to a *failing* `Blocked` phase, so
it counts nothing at all for a stage that yields early, hands back
without a `Blocked` marker, or returns unchanged — and those are exactly
the cases a re-dispatch loop is made of. Count work attempts per stage,
across resumes, using the `attempt <k>` on that stage's `- HANDBACK |`
line for work failures, not `readiness-blocked` diagnostics; a stage that
exhausts three is a blocker. An interactive question
and its answer continuations belong to the same attempt, even when the
host needs a fresh runner to continue it. Three questions must never
exhaust a stage's failure budget.

**An eligible unchanged-artifact work retry spends one work attempt.** It
opens no fourth counter: § *Stage Dispatch* bounds it at one retry,
requires changed inputs or new evidence, and that work retry starts a
new attempt. A readiness diagnostic is not a work retry. The count has
one honest limit
— an unchanged return writes no `- HANDBACK |` line, so nothing records
it on disk and the tally is **in-session only**. A run resumed tomorrow
cannot recover that count; it must still establish changed prerequisites
or new evidence before any retry. Report the missing history as undurable,
in the register the second carve-out in § *The Standing Directive* already
uses, rather than implying a count that survived the session.

**A retry is a diagnose-then-resume dispatch, never a bare
re-dispatch.** `dev-do` § *Iteration Mode (Recovery Path)* resumes a
`Blocked` phase only when the blocker is demonstrably resolved, and
otherwise reports it unchanged — so a bare re-dispatch would burn all
three attempts without ever retrying anything. Attempt *k* hands the
stage sub-agent the recorded `Blocked` reason **and** an explicit
instruction to diagnose and resolve the cause **first**, then resume the
phase. Without that the bound absorbs nothing — not the typo, not the
missing import, not the stale artifact it exists for.

**Each attempt is recorded durably by the stage itself**, using
`dev-do`'s existing log form:

```text
- NOTE | phase: <n> | attempt <k>: <what was tried>
```

That is what makes `plan.md` name the phase that failed *and what was
tried*, rather than leaving it in a transcript that dies with the
session.

**A stage that yields for failure or safety writes `- HANDBACK |`**, in the
form and destination § *The Standing Directive* fixes, before it
returns — so the reason survives the session that produced it, and so a
run resumed tomorrow can see that attempt 1 already failed the same way.
The two carve-outs there are the only exceptions, and the second of them
makes the hand-back **undurable**, which you report as undurable rather
than passing over in silence.

**A blocker is a condition the run cannot proceed *through*.** It is
never merely a question the run would prefer a human answered. These are
blockers:

- a build, test, or lint command the repository's `AGENTS.md` does not
  document;
- a failure the run cannot explain;
- a repository state it cannot safely act on, including anything
  `dev-do`'s pre-flight gate refuses;
- an unexplained unchanged stage return without new diagnostic evidence,
  or a second unchanged return after the single evidence-justified retry
  in § *Stage Dispatch*; a known safety/readiness refusal hands back on
  its first return;
- a `dev-do` **scope-exceeded yield** — non-overridable, marking nothing
  `Blocked` and requiring no `NOTE`, so nothing else in this list would
  catch it;
- a phase or stage that exhausts its three work attempts;
- a missing stage skill file;
- a change the run made to **this** skill.

**On hand-back, report:** the stage and phase, attempts and what each
tried, evidence and decision-record links, whether the
stage's own `- HANDBACK |` line reached disk or the hand-back was
undurable, and **the exact command that resumes the run**. Quote that
command in **full-path form**, never as a bare slot number — a number
re-expands against the date of whatever day the user picks the work back
up, which is not necessarily today. Include the resolved `mode`,
`max_subagents`, and `review_iterations` explicitly so resuming cannot
silently change the run's interaction or budget.

**A user-deferred pause is not a blocker.** Report `awaiting user`, the
pending question and artifact, any undurable answers, and the same
full-path resume command. Preserve work already completed and spend no
failure attempt while the user decides, asks questions, or edits.

## The Review Tail

One **iteration** is a `dev-review` pass **plus** remediation of what it
raised. `review_iterations` is a ceiling, not a target. Stop after the
first cycle unless new material findings, a stated risk, or requested
depth justify another; record that reason in the owning stage's notes.
Never suppress standing findings to meet a budget.

**Scope resolves itself.** `dev-review` derives `plan-slot` scope from
the plan's `COMMIT` entries without asking. When the plan carries **no**
`COMMIT` entries there is nothing to review. Skip the tail, and then
report it in the closing report, naming why.

**Remediation covers Blocker *and* High findings, not Blockers alone.**
That is not a choice this skill gets to make: `dev-plan` § *Iteration
Mode* names Blocker and High as the fold-back set, and `dev-review`
§ *Report Format* prescribes an `review.md` whose `## Next Steps`
section says the same. A narrower rule here would contradict the two
skills this stage dispatches. Lower severities are reported, not
remediated.

**Remediation runs as a `dev-plan` iteration pass** handed
`review.md`, which appends new phases to `plan.md`. Those phases
**keep `dev-plan`'s numbered heading** and carry the marker in the name:

```text
### Phase <n>: Remediation R<k> — <name>
```

`<n>` continues the plan's existing numbering and `<k>` is the 1-based
iteration index. The number is load-bearing: `dev-do`'s `PENDING` /
`COMMIT` / `NOTE` log entries are keyed on `phase: <n>`, so an
unnumbered heading would break the log form and change an existing
skill's output format. A `dev-do` pass then executes those phases under
its ordinary phase-commit protocol.

**Every iteration must leave a marker, clean or not.** A review that
raises no Blocker or High appends no phases, so remediation phases alone
cannot distinguish "iteration *k* ran clean" from "iteration *k* never
ran" — and guessing wrong burns a full two-pass review and overwrites
`review.md` again. The **`dev-plan` remediation stage** therefore
appends one line to `plan.md`'s `## Progress Log` on **every**
iteration, clean or not, before it returns **completed**:

```text
- REVIEW | iteration: <k> | complete
```

An interactive question return is not completion. Write this marker only
after the remediation plan's questions and any execution approval have
been settled, never merely because the stage returned `AWAITING_USER`.

**Iteration *k* is complete when a marker naming `<k>` is present**, and
the tail's progress is the highest `<k>` recorded. Keep `review.md`'s
`## Scope` **Commits** bullet as a cross-check on what the surviving
analysis actually saw — never as the completeness test. Remediation
appends new `COMMIT` entries to the plan by construction, so the two
sets diverge for every iteration that raised a finding, and any test
comparing them is satisfiable only in the clean case. The `| Scope |`
metadata row records a counted description rather than the SHAs
themselves, so it is a cross-check on the count.

Two properties of the marker are load-bearing. It carries **no
`phase: <n>` key**, because a clean iteration appends no phases and so
has no phase number to name, and because a writer that is not `dev-do`
must not inject a phase-keyed entry into a log `dev-do` § *Iteration
Mode (Recovery Path)* reads as recovery evidence. And it **names its
writer explicitly** — the remediation `dev-plan` pass — because "the
remediation pass" does not exist in the clean case the marker was
invented to disambiguate. The form is a fourth labelled entry in a log
`dev-plan` owns and documents, so it extends *that* skill's vocabulary
and changes nothing about `dev-do`'s.

**`dev-review` overwrites `review.md` on each justified new pass.** If
two iterations actually run, the surviving file is the second pass's.
That is `dev-review`'s documented behavior — say in the
closing report which iteration the file reflects.

**Any finding still standing when the budget runs out is reported**
alongside the ledger, and just as prominently. Exhausting the iteration
budget is not a failure and not a hand-back; it is a result the user
reviews.

## Local Efficiency Record

**Measure only what the host actually exposes.** At task start and
accepted handback, use supported private local session/task metadata to
record stage identity, tier, start/end timestamps, acceptance status,
input/output and cache-read/write tokens, AIU, event/agent counts, and
summed model duration where available. Record tool waits, human waits,
interruptions, and parallel overlap separately when observable; unknown
clocks and counters remain `unavailable`, never zero or estimates.
Do not treat summed parallel durations as elapsed time or a green run as
user acceptance. A blocked or awaiting-user handback is not accepted.

Store aggregate counters and IDs only, not prompts, secrets, or tool
payloads. Do not add a telemetry dependency, new artifact, or invented
cache toggle. Keep stable relevant instructions reusable, verify actual
cache telemetry before any authorized cache-setting change, and never
switch models based on unmatched usage aggregates. A routing/model
change needs user approval and a matched quality/cost/time trial.

Compare accepted tasks within the same repository, tier, scope/risk,
and acceptance depth. Track duplicate stages, unchanged-input retries,
authorized changed goals, missing evidence, and reopened High findings
separately. Increasing context/cache writes trigger scope and handoff
inspection, not evidence removal. Report unavailable measurements
honestly; proposed savings are trial targets, not a completion claim.

## Progress Output

Print one short line per stage entered, per artifact written, and per
assumption or user decision recorded. Name the mode at the start.
Automatic runs still have a reader; interactive runs also have question
boundaries, where you relay the stage's prompt and wait.

**Those lines land at stage boundaries**, because that is where you
regain control. The execution stage in particular is **atomic from your
side**: `dev-do` runs every phase and makes every commit before it
returns, so no line of yours can appear between two phases. Neither mode
adds per-phase checkpoints. For those, drive the loop by hand and invoke
`dev-do` with its `checkpoint_every` input. Interactive mode gates the
plan before execution, not each phase inside it.

Keep it to one line each. A stage's own output is that stage's business;
you are reporting the shape of the run, not narrating it.

## Closing Report

In this order:

1. **The slot and mode** — the resolved absolute path and `automatic` or
   `interactive`. Distinguish completion, awaiting user, and a blocker.
2. **The artifacts** — which ones exist, and one line on what each says.
3. **The commits** the execution stage made, SHA and subject, in
   chronological order.
4. **Decisions and evidence** — link the complete assumption ledger by
   artifact and section, give counts and material decisions with their
   rationale, and link readiness/validation receipts. Do not replay
   whole ledgers. Say explicitly when no assumptions were made.
5. **User decisions and pending questions** — report actual user answers
   separately, where the owning stages recorded them, and anything left
   unanswered. Never count a skipped question as an approval.
6. **Standing findings** — anything `dev-review` raised that the
   iteration budget did not close, at the same prominence as the
   decisions, plus which iteration the surviving `review.md` reflects —
   or state that the review tail did not run, and why.
7. **Next steps**, named as available to the **user** and never
   performed: `dev-issue` to publish the request or report and attach
   the plan, `dev-pr-open` to push the branch and open the pull request.
   State plainly that nothing was pushed and no pull request was opened.

Use the compact eight-field handoff in `dev-do` § *Evidence Receipts*
for interrupted/blocked runs. Include the exact full-path resume command,
what remains valid, and one next permitted unblock action. Link detailed
history instead of repeating it; keep unresolved safety constraints
explicit. Include available local aggregate metrics without claiming
unmeasured savings.

## Important Rules

- **This skill owns no artifact.** Every slot file belongs to the skill
  that writes it. Read them all; write none of them. When a stage's
  output is wrong, re-dispatch the stage — never edit its file to fix
  its work.
- **Automatic resolves; interactive asks. A blocker stops both.**
  Default to `automatic` only when the mode is omitted, not when a user
  declines to answer. An interactive question is a pause within the
  stage, not a failure, an approval, or a reason to spend a retry.
- **Never invent a build, test, or lint command.** Commands come from
  the repository's `AGENTS.md`, with the documented fallback to
  `README.md` / `CONTRIBUTING.md` and an obligation to state which
  source was used. A command that is not documented is a blocker, and
  the standing directive never overrides that.
- **Every hand-off and publish offer resolves to *decline*.** Not on the
  merits, not "usually" — always. It is the one class of question the
  standing directive answers with a fixed value, because the alternative
  puts a GitHub write one inference away.
- **Local commits only.** No `git push`, no pull request, no commit
  outside `dev-do`'s phase-commit protocol, and no commit made by you
  directly. Pushing and opening a pull request belong to `dev-pr-open`,
  which the user invokes.
- **No GitHub writes at all**, and `review.md` and `approach*.md` are
  **never** published — not as an issue, not as a comment, not as a
  quotation in a pull request body. Fetching an issue the user named as
  content is a read, and is allowed.
- **Today's date governs slot expansion.** Never reuse a previous day's
  `<MMDD>` for a numeric slot. For an earlier slot the user must give a
  full path — and once resolved, the absolute path is what you use for
  the rest of the run.
- **The concurrency cap is a hard ceiling.** Pass `max_subagents`
  through unchanged and never exceed it. You dispatch one stage
  sub-agent at a time, and it is not counted against the cap.
- **Re-invocation resumes rather than restarts.** There is no flag naming
  a stage to start at; infer it from pending questions and status markers.
  A changed mode affects remaining decisions only. Always include the
  resolved mode in the full-path resume command.
- **Stop and hand back if the run modified this skill.** A stage
  sub-agent reloads its own instructions on the next dispatch, which is
  what makes `dev-do`'s self-modification yield safe. You cannot reload
  *yourself* mid-run. If a completed phase changed `dev-complete`,
  record the durable result and hand back, so that a fresh invocation
  loads the new instructions before anything else runs.
- **Honor repo conventions.** Repository conventions live in
  `AGENTS.md`. Follow its code style and architectural invariants, and
  where it is silent, match the surrounding code rather than importing a
  preference from another repository.

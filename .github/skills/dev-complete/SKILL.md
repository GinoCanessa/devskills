---
name: dev-complete
description: "Drives the entire local inner loop in one invocation, as a conductor over the skills that own each role. USE FOR: carrying a feature request or a bug report from raw input to local commits without re-invoking a skill between stages; resuming a run that handed back. Runs the fixed chain `dev-request` / `dev-report` -> `dev-approach` -> `dev-plan` -> `dev-do`, then a `dev-review` -> `dev-plan` -> `dev-do` remediation tail. Accepts a slot number or a full path to a slot **directory**, a kind of `request` or `report`, the content (prose or an issue reference), and optional `max_subagents` (default 3) and `review_iterations` (default 1; `0` ends the run at `dev-do`). Resolves open questions into recorded assumptions instead of pausing, and reports every one at the close. Owns no artifact. Commits locally only — never pushes, never opens a pull request, never writes to GitHub. Pairs with the skills it drives, and with `dev-issue` / `dev-pr-open`, which stay user-initiated."
---

# Dev Complete Skill

Runs the whole local inner loop under **one** invocation. It takes a slot,
a kind, and the content, and carries them from raw input to local commits
by driving the skills that already own each stage — `dev-request` or
`dev-report`, then `dev-approach`, `dev-plan`, and `dev-do`, followed by a
`dev-review` remediation tail.

This skill is an **orchestrator, not a new role**. It does no PM,
Engineering Lead, Engineer, or QA work itself, it introduces no new
quality bar, and it changes no existing skill's output format, file
ownership, or prompts. Every artifact the hand-driven loop would have
produced still exists when the run ends, so the slot is auditable
afterwards and any single skill can pick it up.

It **owns no artifact.** The one thing it adds is autonomy: where a stage
would stop and ask, it resolves the question on the merits, records the
answer in that stage's own artifact as settled content, and keeps going.
A question never stops the run; a blocker always does. The list of
questions the run answered on the user's behalf is the closing report's
most prominent section, and it is the user's single review point.

## Role

You are the loop's **conductor**. That means:

- You **delegate every role.** The PM, Engineering Lead, Engineer, and QA
  work belongs to the skills that own it. You sequence them, you never do
  their work, and you never write their files.
- You **resolve rather than park.** A question a stage would hand to the
  user is one you answer from the source, the repository, and `AGENTS.md`
  — all of which the stage already has in front of it — and then record.
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

5. **`review_iterations`** *(optional, default `1`)* — how many
   review-and-remediate cycles run after the execution stage. One
   iteration is a full `dev-review` pass **plus** remediation of what it
   raised, so the default leaves an `analysis.md` written *before* its
   own fixes; `2` re-reviews the remediated tree. **`0` skips the review
   tail entirely and ends the run at `dev-do`.** This is the only
   spelling — there is deliberately no flag-style alias for `0`, so that
   there is exactly one name per argument across the whole loop.

## What This Skill Owns

**Nothing.** Every file in the slot is written by the skill that owns it
today, dispatched here as a stage:

- `featurerequest.md` → `dev-request`
- `bugreport.md` → `dev-report`
- `approach-a.md`, `approach-b.md`, `approach-c.md`, and `approach.md` →
  `dev-approach`
- `plan.md` → `dev-plan` (created) and `dev-do` (updated)
- `analysis.md` → `dev-review`

You read all of them and write none of them. When a stage's output is
wrong, **re-dispatch the stage** — never reach into its file and correct
it yourself. Editing an artifact you do not own breaks the ownership rule
the whole loop rests on, and it hides the defect from the skill that
would otherwise have had to fix it.

## The Stage Chain

The chain is **fixed**. No argument skips a stage inside the authoring
chain.

```text
  dev-request / dev-report        the authoring chain: fixed,
            |                     no stage may be skipped
            v
      dev-approach
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

1. Resolve the slot directory to an absolute path and echo it. Resolve
   `SKILLS_SOURCE` and confirm every stage skill file exists.
2. Read the repository's `AGENTS.md` for conventions and commands, with
   the documented fallback to `README.md` / `CONTRIBUTING.md`, and say
   which source you used.
3. Determine the resume point from the artifacts already on disk, and
   rebuild the assumption ledger from them — see § *Resume*.
4. Run each incomplete stage in chain order, one dispatch at a time,
   classifying each outcome from the artifact on disk.
5. Run the review tail `review_iterations` times.
6. Print the closing report.

Three properties of the chain are load-bearing:

- `dev-approach` is **optional in the hand-driven loop and mandatory
  here.** A run nobody is gating is exactly where a wrong solution shape
  would otherwise survive all the way to commits.
- The review tail is the **one sanctioned backward step**. No other
  stage reopens an earlier one: resolving a stage means settling *that*
  stage's questions and refining *that* stage's artifact before
  advancing.
- The chain ends at the review tail's remediation. It never extends to
  `dev-issue` or `dev-pr-open` — publishing and pushing stay
  user-initiated.

## Stage Dispatch

**Resolve the stage skills once.** `SKILLS_SOURCE` is the parent
directory of this skill's own directory — the same resolution
`dev-setup` § *Inputs* uses — and each stage's file is
`<SKILLS_SOURCE>\<skill-name>\SKILL.md`. Confirm every file the run
needs exists before the first dispatch. A missing stage file is a
blocker: hand back and name it.

**Dispatch one sub-agent per stage.** This is load-bearing, not an
implementation detail. It keeps your own context small enough to survive
five-plus stages, it gives the retry loop a clean unit to retry, and —
because a fresh sub-agent re-reads the `SKILL.md` from disk — a
re-dispatch *is* the instruction reload that `dev-do`'s
self-modification yield asks for.

Hand each stage sub-agent exactly four things:

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
   | Review | `dev-review` | `<slot>\analysis.md` |

3. **The standing directive**, in full — see § *The Standing Directive*.
4. **`max_subagents`, unchanged**, when that stage documents the input,
   together with any other input that stage documents and this run
   fixes. The execution stage in particular runs with **no
   checkpointing**, because this run never pauses between phases.

A stage sub-agent runs with the **same model configuration as you**, per
`AGENTS.md` § *Agent guardrails*.

**Classify every stage's outcome from the artifact on disk, never from
the sub-agent's narrative.** A sub-agent that simply stopped is
indistinguishable from one that finished unless the file says so.

- **An authoring stage** is judged by re-reading the artifact's `Status`
  row.
- **The plan stage** is judged by re-reading `plan.md`'s `Status` row.
- **The execution stage** is judged phase by phase: top-level
  `Status: Complete` with every phase `Complete` → advance; any phase
  marked `Blocked` → the diagnose-then-resume path in § *Retry and
  Hand-Back*; phases still `Pending` with no `Blocked` marker → the
  stage yielded early, so re-dispatch it to continue.

Never edit an artifact to fix a stage's work. Re-dispatch the stage.

## The Standing Directive

This is the autonomy contract you hand to **every** stage sub-agent,
alongside the skill path. It changes nothing about the skill's file, its
format, or its prompts — it answers them, for that dispatch only.

It has three parts, and the boundaries between them are the whole point.

### Overridden — resolve on the merits, then record

The stage decides for itself, on the evidence in the source, the
repository, and `AGENTS.md`, and writes the decision into its own
artifact as settled content. It does not ask. This covers:

- The **open-questions walkthrough offer** in `dev-request`,
  `dev-report`, and `dev-plan`. Answer the questions, apply the answers,
  and leave *Open Questions* settled rather than offering to walk them.
- **Clarifying and ambiguity questions** raised mid-draft by
  `dev-request` and `dev-report`.
- **`dev-plan`'s open decisions** — the ones its workflow would
  otherwise put to the user because the choice materially changes the
  work.
- **`dev-approach`'s triviality proposal.** Default: **decline** it and
  run all three authors. A triviality call made before any design work
  is itself a claim about the solution's shape, and nobody is gating
  this run.
- **`dev-approach` § *Re-Invocation Modes*,** which stops and asks
  whenever the slot already holds any `approach*.md`. That fires on
  **every resume into a partly-finished approach stage**, so it must be
  answered rather than waited on. Default: **`judge-only`** when all
  three author files exist and are current with the source, and
  **`regenerate`** when they are missing or stale.
- **`dev-approach` § *User Override*'s** disagree-with-the-judge offer.
  Default: **accept the judge.** The run has no standing to overrule a
  verdict it just commissioned.
- **`dev-review`'s scope prompt**, in the case where it fires at all. It
  does not fire when `plan-slot` scope resolves.

### Always resolved to *decline* — never on the merits

Every **hand-off offer** and every **publish offer**, without exception
and without weighing it:

- the `dev-approach` hand-off that `dev-request` and `dev-report` close
  with;
- the `dev-plan` hand-off that `dev-approach` closes with;
- the `dev-approach` nudge `dev-plan` makes when a slot holds no
  `approach.md`;
- every *"Publish this to GitHub?"* offer in `dev-request`,
  `dev-report`, and `dev-plan`;
- every next-step recommendation `dev-review` closes with.

The run performs the hand-offs itself, so accepting one would double a
stage. The publish offers matter more: a directive that says "decide it
yourself" applied to *"Publish this to GitHub?"* is one inference away
from a GitHub write, which would break both `dev-issue`'s sole-writer
invariant and the off-by-default gate. **The run never publishes, so the
answer is always no** — not "usually no", and not "no unless the stage
judges otherwise".

### Never overridden

- **"Never invent a build or test command."** An undocumented command is
  a **blocker**, not an assumption. It is the one question whose right
  answer is to stop.
- **`dev-do`'s yield conditions 1, 2, 5, and 6** — a blocked phase, a
  decision that materially exceeds the plan's scope, a pre-flight
  inconsistency, and a change to the currently executing skill. These
  are safety gates, not preference prompts. Suppressing condition 5 in
  particular would let an unattended run commit on top of a dirty or
  unexpected tree.
- **Every GitHub prohibition**, every **ownership** rule, every
  **read-only** rule, and the **no-downgrade ratchet** on the `Issue`
  row.

### The shape of a recorded answer

An answer lands in the stage's artifact as **settled content, in the
section it belongs to, written as a decision the artifact made.** Never
as *"the user said"* — the user said nothing. Never as a question left
open. A later reader must be able to see that a choice was made and what
it rested on. Each answer is *additionally* recorded as a ledger line;
see below.

## The Assumption Ledger

Every question the run answered on the user's behalf is recorded, with
five things: the **stage**, the **question**, the **answer chosen**, a
**one-line rationale**, and the **artifact and section** the answer
landed in.

Two rules make the ledger survive a hand-back — which is an *expected*
outcome, and therefore cannot be allowed to lose the run's single review
point.

**1. Every assumption is written to a greppable, named location in the
artifact the owning skill already writes.** The owning stage writes it,
as part of the pass that made the decision; you never write it yourself.
The line carries a fixed prefix, so the whole ledger is one search away:

```text
ASSUMPTION | stage: <stage> | <question> — <answer> (<rationale>)
```

It goes in a named section of that stage's own artifact, following the
labelled-entry convention `plan.md`'s `## Progress Log` already uses:

| Artifact | Section |
|-|-|
| `featurerequest.md` | `## Assumptions` |
| `bugreport.md` | `## Notes` |
| `approach.md` | `## Notes` |
| `plan.md` | `## Notes` |
| `analysis.md` | `## Notes` |

`featurerequest.md` is the only slot artifact with an `## Assumptions`
section; the rest carry a `## Notes` section and no assumptions section,
so `## Notes` is where their ledger lines go. The **prefix**, not the
heading, is what makes the ledger findable.

The ledger line is **in addition to** the settled content the answer
becomes, never a substitute for it. The content is what a reader of the
artifact needs; the line is what the ledger is rebuilt from.

**2. Resume rebuilds the ledger from disk** before continuing, by
reading those sections out of the artifacts that already exist. A
resumed run therefore closes with the assumptions made *before* the
hand-back as well as after.

This is the closing report's most prominent section. It is never
summarized away, never truncated, and never folded into a sentence about
how the run "made some assumptions along the way".

## Resume

A re-invocation with the same arguments **resumes**. It is not a mode
switch and it needs no flag. Pick up at the first incomplete stage,
judged by **status markers, never by file presence** — a file that
exists proves a stage started, not that it finished.

- **Authoring** is complete when the request's or report's `Status` row
  reads `Ready-for-plan`.
- **Approach** is complete when `approach.md` exists and carries a
  `## Selected` section.
- **Plan** is complete when `plan.md`'s `Status` row reads
  `Ready-to-execute` or a later value.
- **Execution** is complete when the plan's top-level `Status` reads
  `Complete` **and** every phase's `**Status:**` reads `Complete`.
- **Review tail** progress is read from `analysis.md` and the plan's
  remediation phases — see § *The Review Tail* for the per-iteration
  test.

A `Blocked` marker anywhere means resume **re-enters** that stage rather
than skipping past it. Rebuild the assumption ledger from the artifacts
before continuing, and say in your first response which stage you
resumed at and why.

## Retry and Hand-Back

**A failing phase gets three attempts in total — the first, plus two
retries.** The counter is **per phase**, not per stage, so one stubborn
phase cannot spend a long plan's whole budget. The bound is internal and
is deliberately **not** a caller knob: a phase that keeps failing is a
wrong phase, and retrying it harder will not make it right.

**A retry is a diagnose-then-resume dispatch, never a bare
re-dispatch.** `dev-do` § *Iteration Mode* resumes a `Blocked` phase
only when the blocker is demonstrably resolved, and otherwise reports it
unchanged — so a bare re-dispatch would burn all three attempts without
ever retrying anything. Attempt *k* hands the stage sub-agent the
recorded `Blocked` reason **and** an explicit instruction to diagnose
and resolve the cause **first**, then resume the phase. Without that the
bound absorbs nothing — not the typo, not the missing import, not the
stale artifact it exists for.

**Each attempt is recorded durably by the stage itself**, using
`dev-do`'s existing log form:

```text
- NOTE | phase: <n> | attempt <k>: <what was tried>
```

That is what makes `plan.md` name the phase that failed *and what was
tried*, rather than leaving it in a transcript that dies with the
session.

**A blocker is a condition the run cannot proceed *through*.** It is
never merely a question the run would prefer a human answered. These are
blockers:

- a build, test, or lint command the repository's `AGENTS.md` does not
  document;
- a failure the run cannot explain;
- a repository state it cannot safely act on, including anything
  `dev-do`'s pre-flight gate refuses;
- a phase that exhausts its three attempts;
- a missing stage skill file;
- a change the run made to **this** skill.

**On hand-back, report:** the stage and the phase, the attempts and what
each one tried, the evidence, the ledger rebuilt so far, and **the exact
command that resumes the run**. Quote that command in **full-path
form**, never as a bare slot number — a number re-expands against the
date of whatever day the user picks the work back up, which is not
necessarily today.

## The Review Tail

One **iteration** is a `dev-review` pass **plus** remediation of what it
raised. `review_iterations` counts iterations, not passes.

**Scope resolves itself.** `dev-review` derives `plan-slot` scope from
the plan's `COMMIT` entries without asking. When the plan carries **no**
`COMMIT` entries there is nothing to review: skip the tail, record that
you skipped it and why, and say so in the closing report.

**Remediation covers Blocker *and* High findings, not Blockers alone.**
That is not a choice this skill gets to make: `dev-plan` § *Iteration
Mode* and `dev-review` § *Next Steps* both name Blocker and High as the
fold-back set, and a narrower rule here would contradict the two skills
this stage dispatches. Lower severities are reported, not remediated.

**Remediation runs as a `dev-plan` iteration pass** handed
`analysis.md`, which appends new phases to `plan.md`. Those phases
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

**A clean iteration must still leave a marker.** A review that raises no
Blocker or High appends no phases, so remediation phases alone cannot
distinguish "iteration *k* ran clean" from "iteration *k* never ran" —
and guessing wrong burns a full two-pass review and overwrites
`analysis.md` again. Read `analysis.md`'s `## Scope` section instead:
its **Commits** bullet lists the SHAs that were reviewed. **Iteration
*k* is complete when that commit set equals the plan's current set of
`COMMIT` entries.** The `| Scope |` metadata row records a counted
description rather than the SHAs themselves, so it is a cross-check on
the count, not the test.

**`dev-review` overwrites `analysis.md` on every pass.** With
`review_iterations: 2` the surviving file is the second pass's. That is
`dev-review`'s documented behavior and is not changed here — say in the
closing report which iteration the file reflects.

**Any finding still standing when the budget runs out is reported**
alongside the ledger, and just as prominently. Exhausting the iteration
budget is not a failure and not a hand-back; it is a result the user
reviews.

## Progress Output

Print one short line per stage entered, per artifact written, and per
assumption recorded. The user who invoked this skill is watching the
session even though they are not answering prompts, and the progress
output is their only chance to interrupt a run they disagree with before
it reaches commits. Write it for a reader.

Keep it to one line each. A stage's own output is that stage's business;
you are reporting the shape of the run, not narrating it.

## Closing Report

In this order:

1. **The slot** — the resolved absolute path.
2. **The artifacts** — which ones exist, and one line on what each says.
3. **The commits** the execution stage made, SHA and subject, in
   chronological order.
4. **The assumption ledger** — every question the run answered on the
   user's behalf, in full: one entry per question, with the stage, the
   answer, the rationale, and where it landed. **This is the most
   prominent part of the report, and the one part that must not be
   summarized away.** Because the run never paused, it is the user's
   single review point.
5. **Standing findings** — anything `dev-review` raised that the
   iteration budget did not close, at the same prominence as the
   ledger, plus which iteration the surviving `analysis.md` reflects.
6. **Next steps**, named as available to the **user** and never
   performed: `dev-issue` to publish the request or report and attach
   the plan, `dev-pr-open` to push the branch and open the pull request.
   State plainly that nothing was pushed and no pull request was opened.

## Important Rules

- **This skill owns no artifact.** Every slot file belongs to the skill
  that writes it. Read them all; write none of them. When a stage's
  output is wrong, re-dispatch the stage — never edit its file to fix
  its work.
- **A question never stops the run. A blocker always does.** Resolving a
  question into a recorded assumption is the entire point of the skill.
  Confusing the two defeats it in both directions: stopping on a
  question makes the run no better than the hand-driven loop, and
  proceeding through a blocker produces work nobody can trust.
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
- **No GitHub writes at all**, and `analysis.md` and `approach*.md` are
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
- **Re-invocation is the recovery path, not a mode switch.** The same
  command resumes a handed-back run. There is no flag naming a stage to
  start at; resume is inferred from the artifacts' status markers.
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

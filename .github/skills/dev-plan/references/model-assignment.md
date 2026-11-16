# Model Assignment Contract

Resolve model selection before dispatch, independently of whether a task
warrants delegation. Read the target's `AGENTS.md` under
`## Agent guardrails` / `### Subagent model policy`. If `AGENTS.md` is
absent, use the skill's documented `README.md` / `CONTRIBUTING.md`
fallback and state the source. An absent or unreadable policy means
`uniform`; report that fallback. A present unknown policy or an incomplete
explicit assignment is a blocker, not permission to fall back.

## Policies

- **`uniform` means the same configuration.** Every child inherits the
  spawning agent's model and applicable model options. Do not rely on a
  built-in agent's cheaper default when that would change the selection.
- **`tiered` preserves the two-tier policy.** Reasoning roles inherit the
  spawning agent's configuration; mechanical roles use the recorded
  `Mechanical-tier model`. A built-in is suitable only when its effective
  selection satisfies that policy; its name is not proof of its model.
- **`role-based` selects by task.** Normalize `role` to `role-based`.
  Read the `Role assignments` table and resolve the rules below. Tier
  classifications still describe the work; they do not override an
  explicit role assignment.
- **`auto` delegates selection to the harness.** Use its documented
  automatic-selection mechanism, not a model chosen by the skill, the
  mechanical tier, or an assumed inheritance default.

An explicit user assignment for this task overrides repository settings
for that scope only. Record its source; never rewrite `AGENTS.md` as a
side effect of dispatch. Preserve legacy per-run overrides where the
calling skill documents them; reject incompatible selectors.

## Role Assignments

Use this target-configured table beneath the policy settings:

```markdown
#### Role assignments

| Role | Selection |
|-|-|
| orchestration | <model-id> |
| planning | <model-id> |
| judging | <model-id> |
| review | <model-id>, <another-model-id> |
| engineering | <model-id> |
| documentation | <model-id> |
| discovery | inherit |
| verification | auto |
| default | auto |
```

Rows other than `default` are optional. `default` is required under
`role-based`. This is a schema example, not a proposed assignment.
Selections are an available model ID with any explicitly confirmed
supported options, `inherit`, or `auto`. `inherit` means the spawning
configuration; `auto` has the same harness semantics as the policy.
Allow multiple distinct selections **only for `review`**, and require
concrete model IDs there so each requested model is identifiable.
Reject duplicate rows, conflicting options, unknown roles, unresolved
names, and lists for single-owner work. Do not guess which model a
friendly name denotes; resolve it against current harness capabilities.

**Resolve specific work before its domain, then use the default.**

1. Match the task role: `orchestration`, `planning`, `judging`, `review`,
   `discovery`, or `verification`.
2. If no such row exists, match its work domain: engineering or
   documentation. Review of documentation falls back to documentation;
   review of application changes falls back to engineering. For genuinely
   mixed work without a primary domain, proceed to `default`.
3. Use `default` if neither matches. Never infer an unrecorded model.

`review` includes code/QA/documentation review and adversarial critiques.
`planning` includes approach authors and plan authors; `judging` is the
selection between alternatives. `engineering` is implementation or
debugging; documentation is intake writing, source synthesis, and
documentation delivery. Discovery and command execution retain their own
roles even when performed inside planning, engineering, or review.
Publishing has neither domain and uses `default` when no explicit
task-scoped user assignment applies.

**The domain follows the deliverable, not its file format.** A plan for
application changes is engineering work even when written in Markdown;
a plan for documentation-only delivery has the documentation domain.
Orchestration itself has neither domain and falls back to `default`
without an orchestration row.

**Classify the work, not the agent wrapper.** A stage runner executing a
plan is planning; one executing implementation is engineering, or
documentation for a documentation-only phase. An approach coordinator is
planning. A review-stage coordinator is orchestration: it owns one report
and dispatches the selected read-only reviewers. Never run multiple
artifact-writing coordinators to satisfy a review-model list. A fallback
agent retains the original task role, domain, and resolved selection.

## Harness Resolution and Running Callers

Inspect the current harness's supported dispatch controls and available
models before accepting assignments. Bind provider-qualified identities
when the catalog requires them; preserve confirmed reasoning or other
model options only where supported. Do not invent a model ID, provider,
option, or automatic-selection API.
Do not carry model-specific options to another selection unless they
were explicitly confirmed and are supported there.

For `auto`, use a native automatic selector when available. Omit a model
field only when the harness documents that omission requests automatic
selection; omission that inherits a fixed parent model is **not** auto.
An automatic permission, approval, or execution mode is not automatic
model selection.
Agent-definition or built-in defaults must not override the requested
selection. If an agent API cannot express it, use an available supported
dispatch mechanism that preserves its role and tool restrictions, or stop
with the unsupported capability. Never silently substitute a model,
provider, policy, or less restrictive agent.

**Configuration cannot change a running caller.** Apply orchestration
and other in-process assignments when launching that caller through the
harness. If a required assignment differs from the current caller, report
the mismatch and request a correctly configured invocation, or use a
supported user-authorized single-owner handoff. Do not claim a switch
occurred, spawn a duplicate owner, or change global harness defaults.
Under auto, distinguish a caller launched with automatic selection from
children whose selection is still to be resolved.

Never put a `model:` pin in a portable agent definition. Supply resolved
selections through runtime controls; retain its role brief and tool
restrictions, including read-only review and no-delegation constraints.

## Multiple Review Models and Budgets

**Every selected review model independently covers each required scope.**
First choose the skill's scope/track branch. Then expand each required
reviewer invocation once per selected model. A small combined review with
two models needs two independent combined reviewers. Separate engineering
and QA tracks with two models need four invocations, not one model per
track. Do not share findings until synthesis, collapse the models into a
single prompt, or choose just the first model. Only the original owning
skill writes the final report and merges duplicate findings.

Preserve concurrency and total-child ceilings. Serialize to fit
concurrency; every selected-model invocation, failure, retry, and recheck
counts toward existing budgets. Reserve all mandatory model/track
combinations and planned rechecks before optional discovery. If they
cannot fit the total budget, surface the required count and request an
explicit budget or scope decision before dispatch; never raise a ceiling
or drop a requested model silently. A retry or recheck preserves the
affected model assignment and scope.

## Evidence and Resume

Record the effective policy, task role/domain, selection, configuration
source, and any automatic-selection mechanism in the owning skill's
existing options/identity or evidence records. Record the actual model
only when the harness exposes it; otherwise mark it unavailable, not
inherited or guessed. Keep private runtime identifiers in private
session metadata.

Include effective assignments in relevant stage and reviewer identities.
A policy change alone does not replay unaffected work; a changed
assignment invalidates reuse of the affected review/dispatch evidence.
Resume known active owners rather than starting duplicates. Model
selection never grants extra writes, delegation, publishing, or budget.

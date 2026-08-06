# AGENTS.md

Canonical, machine-readable conventions for automated agents working in
**{TBD: repository name}**. This file is the single source of truth that the
`.github/skills/dev-*` skills read before naming any build, test, or lint
command.

**Precedence.** This file is authoritative for commands, conventions, and
invariants an agent must follow. [`README.md`](README.md) is authoritative
for rationale, configuration reference, and operational detail, and is the
place to look for the "why". If this file contradicts the repository itself,
the repository wins — fix this file.

---

## What this repository is

{TBD: 1–3 sentences. What the project does, who consumes it, and the one
framing fact that settles arguments — e.g. "this is a port of X, so 'what
does X do?' is usually the deciding argument", or "this ships as a NuGet
library, so public API changes are breaking changes".}

---

## Repository layout

| Path | Contents |
|-|-|
| {TBD: `path/`} | {TBD: what lives here} |

{TBD: note which paths are gitignored, and where committed assets must live
if that is constrained.}

---

## Toolchain pins

- {TBD: SDK / runtime / compiler versions and where they are pinned
  (`global.json`, `.tool-versions`, `rust-toolchain.toml`, `.nvmrc`,
  `pom.xml` properties, …). Say whether the pin is exact or a floor.}
- {TBD: language version, nullability / strictness settings, and where they
  are set (`Directory.Build.props`, `tsconfig.json`, `pyproject.toml`, …).}
- {TBD: dependency-version strategy — e.g. central package management, lock
  files, workspace catalogs — and the rule that follows from it.}
- {TBD: local tools that must be restored before certain work.}

{TBD: if warnings are errors, say so here in its own subsection and name the
only sanctioned suppressions.}

---

## Build

```{TBD: shell}
{TBD: full build command}
```

Scoped to a single {TBD: project / module / package}:

```{TBD: shell}
{TBD: scoped build command}
```

The expected baseline is {TBD: e.g. **0 warnings, 0 errors**}. If you see
anything else, investigate before attributing it — it may be environmental
or pre-existing. Confirm against a clean checkout or `HEAD` before calling
it a regression.

{TBD: if the repository has more than one build track (native + managed,
per-platform, publish-only checks such as AOT/ILC), give each track its own
subsection, say what each one catches that the others cannot, and say which
host platforms can run it.}

---

## Test

{TBD: name the test framework(s) and runner(s), and state explicitly which
filter syntax is valid — this is the single most common thing an agent gets
wrong.}

### Full suite

```{TBD: shell}
{TBD: full test command}
```

### Scoped — one {TBD: project / module / package}

```{TBD: shell}
{TBD: scoped test command}
```

### Focused — one class or one test

```{TBD: shell}
{TBD: focused test command}
```

**Prefer the smallest command that covers the change.** Escalate to the full
suite only when the focused run indicates you need to.

{TBD: document any gate that needs setup — hermetic caches, fixtures,
environment variables, services — with the exact setup commands. A
verification step nobody can run is not a verification step.}

---

## Lint / format

```{TBD: shell}
{TBD: lint or format command, or "No separate lint step; the build enforces
this." Do not invent one.}
```

---

## Run

```{TBD: shell}
{TBD: how to run the application / CLI / dev server locally, including any
required configuration or environment variables}
```

---

## Code style

{TBD: name the authoritative source (`.editorconfig`, `.prettierrc`,
`ruff.toml`, `rustfmt.toml`, …) and whether it is enforced by the build or
only by the editor / a formatter command.}

- {TBD: encoding, line endings, final newline, indent width per file type.}
- {TBD: the handful of rules an agent must not get wrong. Be explicit about
  rules that differ from the wider ecosystem default, and be explicit where
  the repository deliberately has **no** opinion — "X is idiomatic here, do
  not raise review findings about it" is as valuable as a rule.}
- Match the surrounding file. Consistency with neighbouring code beats any
  general preference.

### Architectural invariants

These are decisions, not preferences. Violating one is a review Blocker.

- {TBD: each invariant, why it exists, and what the sanctioned escape hatch
  is (if any). Examples of the shape: allowed dependency direction between
  projects, serialization strategy, where committed assets live, what the
  process may and may not do at startup.}

---

## Commit conventions

- **Conventional commits**: `<type>(<scope>): <subject>` using {TBD: the
  sanctioned type list}. Subject in the imperative, target ≤ 72 characters.
  Scope is {TBD: optional but encouraged / required / unused}.
- {TBD: required commit trailers, verbatim}
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
- `scratch/` is **ignored** ({TBD: `/scratch` in `.gitignore` /
  `.git/info/exclude`}). Nothing in it is ever committed.
- Because the slot is ignored, **no plan phase may declare a `scratch/` path
  as an owned path.** `plan.md` is a control file that `dev-do` edits
  continuously and never stages or commits.

---

## Agent guardrails

- Read this file before proposing any build, test, or lint command. **Never
  invent a command.** If something you need is not documented here, say so
  rather than guessing.
- Subagents must use the same model configuration as the spawning agent.
- Do not add new linting, building, or testing tooling without being asked.
- Prefer the smallest targeted verification that covers the change; escalate
  to the full suite only when the targeted run indicates it is needed.
- {TBD: repository-specific guardrails — ledger files that must stay
  current, directories that are off-limits, generated code that must not be
  hand-edited, external trees the agent is expected to consult.}

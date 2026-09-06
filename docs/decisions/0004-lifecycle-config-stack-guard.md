# ADR-0004: Project lifecycle config replaces the hard-coded no-app-stack guard

**Date:** 2026-09-06
**Status:** accepted
**Deciders:** App-Factory maintainers; implemented by Arena Agent Mode

## Context

The source foundation this template was derived from blocked application-stack
artifacts — `package.json`, `Dockerfile`, `src/`, and similar — while no product
decision existed. The mechanism was a constant near the top of the gate:

```bash
ALLOW_APP_STACK="${ALLOW_APP_STACK:-0}"
```

That was correct for a single repository whose product was permanently
undefined. It cannot survive in a reusable factory. Every repository generated
from this template is *expected* to acquire a stack eventually, and the only
documented way to permit one was to edit `scripts/verify.sh` — the very script
whose job is to be trustworthy. An environment-variable override made it worse:
`ALLOW_APP_STACK=1 bash scripts/verify.sh` would stand the guard down for a
single run, leaving no trace in the repository at all.

A guard that is disabled by editing the guard, or by an untracked environment
variable, is not a control. The state it encodes — "has this project chosen a
stack yet?" — is a property of the *project*, not of the checker.

## Decision

Move lifecycle state out of the gate and into committed configuration at
`config/project.env`:

```text
PROJECT_PHASE=factory|discovery|architecture|implementation
ALLOW_APP_STACK=0|1
STACK_DECISION_ADR=docs/decisions/NNNN-<title>.md
```

`scripts/verify.sh` gains a `lifecycle` check that validates the file's shape
and — critically — its internal consistency. `ALLOW_APP_STACK=1` is accepted
only when `PROJECT_PHASE=implementation` **and** `STACK_DECISION_ADR` names a
file that actually exists under `docs/decisions/`. `check_no_app_stack` then
reads that validated state instead of a constant, and reports `SKIP` rather
than `PASS` when the guard stands down.

The file is parsed line-by-line and never sourced, so a malformed or hostile
value cannot execute in the gate that inspects it. There is no environment
override: the state is whatever is committed.

## Alternatives considered

### Alternative: Keep the constant in `verify.sh` and document editing it

- **Pros:** Zero new files; exactly what the source foundation did.
- **Cons:** Graduating a project means editing the quality gate. Reviewers must
  distinguish a legitimate one-line flip from a hostile weakening in the same
  file that contains every other check. Nothing forces an ADR to exist.
- **Why not:** The gate must be the least-edited file in the repository. Making
  routine lifecycle progress require surgery on it trains everyone — human and
  agent — to treat gate edits as normal.

### Alternative: Keep the `${ALLOW_APP_STACK:-0}` environment override

- **Pros:** Convenient for local experimentation.
- **Cons:** Leaves no trace in the repository. CI and a developer can disagree
  about whether the guard is up, and the PR diff shows nothing.
- **Why not:** State that governs a security-relevant guard must be committed
  and reviewable. Convenience here buys an invisible bypass.

### Alternative: Infer the phase from repository contents

Detect a `package.json` and conclude the project is in implementation.

- **Pros:** No configuration to maintain.
- **Cons:** Circular: the guard exists precisely to catch an unintended
  `package.json`. Inferring intent from the artifact makes the check
  self-defeating.
- **Why not:** It converts every accident into a decision.

### Alternative: Derive the phase from the ADR directory alone

- **Pros:** One source of truth; no duplicated state.
- **Cons:** Requires parsing ADR prose to decide whether a stack was chosen and
  accepted. Brittle, and easy to trip with an ADR that merely *discusses* a
  stack.
- **Why not:** An explicit pointer (`STACK_DECISION_ADR`) plus an existence
  check is unambiguous, and it still requires the ADR to exist.

## Consequences

### Positive

- The transition is a small, reviewable config diff with an ADR attached, not
  an edit to the gate.
- The guard cannot be stood down silently: three values must agree, and the
  referenced ADR must exist on disk.
- Both directions are provable, and are proved: `scripts/selftest.sh` asserts
  the guard rejects stack artifacts before the transition, stands down after a
  valid one, and still rejects `ALLOW_APP_STACK=1` when the phase or the ADR is
  missing or wrong.
- Documentation, CI, and the gate all read the same state, so `docs/ROADMAP.md`
  and repository reality cannot drift.
- The template is genuinely reusable: a new project graduates without ever
  touching `scripts/verify.sh`.

### Negative

- One more file to keep valid, and one more check to maintain.
- Lifecycle state is duplicated conceptually between `config/project.env` and
  `docs/ROADMAP.md` prose; the config is authoritative and the roadmap must
  follow it.
- Standing the foundation guard down leaves a real gap until stack-specific
  lint/test/build gates are added. The `SKIP` message says so explicitly, but
  it is a genuine window that a project must close deliberately.

### Follow-ups

- When a project reaches `implementation`, add stack-specific CI jobs
  *alongside* `Foundation gate` and `Independent checks`, never replacing them.
- Consider a future check that warns when `PROJECT_PHASE=implementation` but no
  stack-specific CI job exists, closing the window described above.

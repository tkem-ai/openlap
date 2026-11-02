# Copilot Instructions for Open Lap

Project conventions (stack, commands, code style, architecture constraints, boundaries
and git workflow) live in [`AGENTS.md`](../AGENTS.md). This file adds only the code
review process.

## Code Review

- Reviews **amend `.github/copilot-review.md` in place** (header: date, commit, build
  status, tests, one *Verified* paragraph); never new files, dated sections or a
  per-commit change summary (git history is the record).
- Findings are tables ordered High, Medium, Low, Info with columns
  `# | Priority | Location | Finding`; renumber after changes; cross-reference as
  "Code #3". **Resolved issues are removed**, never marked resolved.
- Keep it brief: one or two sentences per row, numbers only where they decide
  something, current snapshot only (no comparisons with earlier releases). Record
  settled matters under *Maintainer Decisions*. A review changes nothing; fix only
  when asked.
- Do not flag the conventions in `AGENTS.md` (loose equality, legacy template syntax,
  NgModules, split `rxjs` imports, missing tests, `strictNullChecks: false`
  consequences unless they cause a bug). Do flag protocol or framing errors,
  reconnect semantics, RxJS side effects, subscription leaks, unguarded
  `backendFilter`, translation key drift, and unrelated reformatting or version
  bumps in a change.

## Maintainer Decisions (do not re-raise)

- Loose equality and legacy template syntax stay; they are conventions, not defects.
- No automated tests or lint; hardware behaviour is verified manually on a CU.
- `ControlUnit.data` keeps `resetOnDisconnect: false`; `status` and `getState()` keep
  `shareReplay({ bufferSize: 1, refCount: false })` so late subscribers get the last
  value.
- The reconnect `delay` callback deliberately does not log the error; state
  transitions are the signal.
- `Session.createGrid()` keeps an unbounded `shareReplay({ refCount: true })` so late
  subscribers replay lap history.
- `this.finish(id)` stays inside `Session`'s `scan()` until it can be moved and tested
  on hardware in one go.

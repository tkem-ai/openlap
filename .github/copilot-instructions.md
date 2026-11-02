# Copilot instructions for Open Lap

The single source of truth for agents in this repository is [`AGENTS.md`](../AGENTS.md)
at the repository root. Read it first and follow it for architecture, code style,
build/verification steps, i18n rules and git conventions. Keep the two files in
sync: if you change a convention, update `AGENTS.md`, not this file.

## Review focus

When reviewing pull requests, prioritise:

- **Hardware/protocol correctness** in `src/app/carrera/`, `src/app/backend/`,
  `src/app/rms/session.ts` and `src/app/settings/firmware.page.ts`: command
  framing, CRCs, reconnect semantics (`connectable` must keep
  `resetOnDisconnect: false`), and side effects in RxJS pipelines.
- **Subscription leaks**: components must unsubscribe in `ngOnDestroy`.
- **`backendFilter` guards**: it may be `undefined`; unguarded string calls
  break the observable.
- **Translation parity**: every file in `src/assets/i18n/` must keep the exact
  key set of `en.json`; placeholders and product names stay untranslated.
- **Scope creep**: unrelated reformatting, version bumps or CHANGELOG edits in a
  functional change.

## Do not flag

- Loose equality (`==` / `!=`), legacy template syntax (`*ngIf`, `*ngFor`),
  NgModule/`standalone: false` usage, or separate `rxjs` / `rxjs/operators`
  imports. These are deliberate project conventions.
- Missing unit tests. The project has no test suite by design; ask for a
  hardware test plan instead.
- `strictNullChecks: false` / `noImplicitAny: false` consequences, unless they
  cause an actual bug.

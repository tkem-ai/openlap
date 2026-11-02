# Code Review — Open Lap

Point-in-time review of the `master` branch as of October 2026 (after
`6d8be65 Modernize reconnect handling.` and the firmware-update work). Earlier
findings are kept below with their current status so the history stays useful.

## Overall quality: A-

The codebase is well-structured for a mid-size Angular/Ionic app. The domain
layer (`carrera/`) is clean, decoupled from Angular, and now free of deprecated
RxJS operators. Feature modules are logically organised, naming is consistent,
and barrel exports keep imports tidy. The reactive race logic in `Session` and
`RmsPage` is idiomatic and handles real-time state well.

**Strengths**

- Clean separation between hardware abstraction (`ControlUnit` / `Peripheral`),
  session logic and UI.
- `publish()` / `retryWhen()` replaced by `connectable()` and
  `retry({ delay, resetOnSuccess })`; `shareReplay` buffers are explicit and
  `refCount: false` is documented as intentional.
- Consistent `OnDestroy` cleanup across components.
- Full i18n coverage with natural-language keys; all nine locale files share an
  identical key set (116 keys).
- `NavParams` fully removed; popovers and modals use `@Input()` + `componentProps`.
- Firmware update is isolated in `FirmwarePage` with a `CanDeactivate` guard.

**Weaknesses**

- One remaining side effect inside `scan()` in timing-critical session code.
- The firmware-update state machine has no error or timeout path, so a dropped
  link during an update leaves the page stuck in a "busy" state.
- No automated tests or lint; all hardware paths are verified manually only.

---

## HIGH

### 1. Deprecated RxJS operators `publish()` / `retryWhen()` — **RESOLVED**

**File:** `src/app/carrera/control-unit.ts`

Fixed in `6d8be65`. `ControlUnit.data` now uses
`connectable(…, { connector: () => new Subject(), resetOnDisconnect: false })`
and `retry({ resetOnSuccess: true, delay })`. The backoff is computed from
RxJS's own `retryCount` in `getReconnectDelay()`, so the former `scan()`-based
counter and its reliance on `state.value` are gone. Nothing further to do;
keep `resetOnDisconnect: false` (see `AGENTS.md`).

### 2. Stateful `doReconnect()` reading `state.value` — **OBSOLETE**

Superseded by #1. `doReconnect()` no longer exists; `getReconnectDelay(retryCount)`
is pure and `state` transitions happen in the `retry` delay callback.

---

## MEDIUM

### 3. `scan()` side effect calls `this.finish(id)` — **OPEN**

**File:** `src/app/rms/session.ts` (inside `createGrid()`, marked with a `TODO`)

The accumulator invokes `this.finish(id)`, which mutates `this.mask`, sends CU
commands and pushes `finished`. Side effects belong in a downstream `tap()`.
The accumulator already carries `finished` as the fourth tuple element, so the
`false → true` edge can be detected after the `scan()`:

```ts
scan(([times, last, best, finished]: TimeInfo, [id, time, sensor]) => {
  // … existing accumulation; only set `finished = true` here …
  return <TimeInfo>[times, last, best, finished];
}, <TimeInfo>[[], [], [], false]),
distinctUntilChanged((a, b) => a[3] === b[3]),
tap(([, , , finished]) => {
  if (finished) {
    this.finish(group.key);  // fires exactly once per driver
  }
})
```

**Verify when migrating**

- `finish()` must fire once per driver; `distinctUntilChanged` on the flag (or
  `pairwise()`) prevents resending `setMask` / `setFinished` on every later lap.
- `group.key` equals the `id` previously passed (`groupBy(([id]) => id)`).
- Test a multi-car session run to its lap and time limits on hardware; the
  checkered flag and pace-car behaviour must be unchanged.

### 4. Firmware update has no error/timeout handling — **NEW**

**File:** `src/app/settings/firmware.page.ts`, `startFirmwareUpdate()` / `update()`

The raw connection is consumed with `.subscribe(value => this.update(value))`
and no error handler, and there is no timeout between responses. If the BLE or
serial link errors or the CU stops answering mid-update:

- the error is unhandled (surfacing only via RxJS's global handler), and
- `status` stays `'updating'` / `'finishing'`, so `busy` remains `true` and
  `firmwareCanDeactivateGuard` blocks the user from leaving the page.

**Suggested fix:** add an `error` callback (and ideally a `timeout()` operator on
the response pipeline) that logs via `LoggingService`, resets `status` to
`'init'`, unsubscribes, and attempts `cu.reconnect()`. Consider also an explicit
"Abort" affordance in the template for the `'starting'` state, where nothing has
been written yet. A failed update is a rare but high-impact event for the user,
so this is worth a hardware-verified fix.

---

## LOW

### 5. Minor tidy-ups in `firmware.page.ts`

- `private requests = [];` is implicitly `any[]`; type it as `DataView[]`.
- `confirmFirmwareUpdate()` is `async` but does not `await` `alert.show()`.
- `ngOnInit()` is an empty placeholder.
- The firmware-file sanity check (800–1000 lines, `.hmf` extension) is a
  heuristic; the source `TODO` already notes this.

None of these affect behaviour.

---

## Summary

| Severity | Status   | Finding |
| -------- | -------- | ------- |
| HIGH     | resolved | #1 deprecated `publish()` / `retryWhen()` |
| HIGH     | obsolete | #2 stateful `doReconnect()` |
| MEDIUM   | open     | #3 `scan()` side effect in `Session` |
| MEDIUM   | open     | #4 firmware update lacks error/timeout path |
| LOW      | open     | #5 small cleanups in `firmware.page.ts` |

### Recommended order of work

1. **#4** — add error/timeout handling to the firmware update; it is the only
   path that can strand the user.
2. **#3** — move `finish()` out of `scan()` once a hardware test session is
   available; mechanical change, but timing-critical.
3. **#5** — opportunistically, when next touching `firmware.page.ts`.

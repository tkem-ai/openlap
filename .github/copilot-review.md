# Code Review — Open Lap 2.7.2

**Date:** 2026-11-02
**Reviewed:** `master` at `be68439` ("Switch interval to monthly.", `config.xml`
version 2.7.2, changelog dated 2026-08-07).

**Build status:** `npm run build` passes locally on Node 22.22.0 / Angular 20.3
(one CommonJS warning for `localforage` via `@ionic/storage`). GitHub Actions not
checked.

**Tests:** none; the project has no test suite or linter by design
(*Maintainer Decisions*).

**Verified:** translation key parity of all nine locale files against `en.json`
(116 keys each); the reconnect path (`connectable` / `retry`) and the firmware
update state machine by reading `control-unit.ts`, `ble.ts`, `web-bluetooth.ts`,
`web-serial.ts` and `firmware.page.ts`; every finding below against the source.
Nothing was run on hardware.

## Code — Potential Issues

| # | Priority | Location | Finding |
|---|----------|----------|---------|
| 1 | High | `app.component.ts` constructor | `window.screen.orientation.addEventListener(…)` runs before the `window.screen.orientation && …` guard that follows it. On WebKit before 16.4 (`deployment-target` is 13.0) `screen.orientation` is undefined, the root constructor throws and the app never bootstraps. Move the guard in front. |
| 2 | Medium | `ble.ts` `createObservable()` error branch | A disconnect delivered by the plugin as a non-`Error` object while `isConnected` is true calls `subscriber.complete()`. `retry()` only resubscribes on errors, so `ControlUnit.data` completes: `state` never becomes `'disconnected'`, polling stops, and the `connectable` subject (`resetOnDisconnect: false`) is dead for later `connect()`/`reconnect()`. Usually masked by the request timeout erroring first; verify on hardware (switch phone Bluetooth off mid-race) and consider `subscriber.error(…)`. |
| 3 | Medium | `settings.service.ts` `observe()` | A failed `storage.get()` puts the cached `ReplaySubject` into the error state and leaves it in `subjects`, so `observe(key)` (and `getOptions()`, `getConnection()`, …) stays errored for the app lifetime and `set()` becomes a no-op. A rejected `storage.create()` is not handled at all. |
| 4 | Medium | `session.ts` `toggleYellowFlag()` / `finish()` | Clearing the yellow flag restores `realMask`, captured before any `finish()` that ran during the flag (time limit, `stop()`), so `cu.setMask()` re-enables lap counting for finished cars and the per-car `scan()` keeps accumulating laps. Merge the finish bits into `realMask` in `finish()`. |
| 5 | Medium | `firmware.page.ts` `startFirmwareUpdate()` / `update()` | The raw connection is subscribed without an error handler and no timeout between responses. If the link drops or the CU stops answering, `status` stays `'updating'`/`'finishing'`, `busy` stays true and `firmwareCanDeactivateGuard` traps the user on the page. Add an error path that logs, resets `status` to `'init'`, unsubscribes and reconnects, plus a response timeout. |
| 6 | Low | `session.ts` timer `tap()` | After the countdown reaches 0 every 500 ms tick satisfies `time == 0`, so `finish()` and `finished.next(true)` run on each tick for the rest of the session; hidden by `distinctUntilChanged` downstream. Guard with `!this.finished.value`. |
| 7 | Low | `rms.page.ts` constructor (`pitlane`, `start`) | `mergeMap(cu => cu.getMode()/getStart())` keeps the inner subscription of every superseded `ControlUnit` alive (its `status` never completes), leaking one per device swap and interleaving stale values. `TuningPage` uses `switchMap`, which is correct here. |
| 8 | Low | `rms.page.ts` `ngOnInit()` / `startSession()` | When `cu` becomes `null` the page nulls `session` but leaves `sessionSubscription` running; the nested `firstValueFrom()` chains have no `.catch` and may call `toggleStart()`/`session.start()` on a session that `restartSession()` already replaced. |
| 9 | Low | `control-unit.ts` `disconnect()`, `getVersion()`; `menu.component.ts` `getVersion()` | `disconnect()` unsubscribes but never pushes `'disconnected'`, so `getState()` keeps reporting `'connected'`. `getVersion()` caches a `firstValueFrom` promise that rejects with `EmptyError` if `data` completes (Code #2); the menu feeds it to `version \| async` uncaught. |
| 10 | Low | `menu.component.ts` `onMenuOpen()` | The user-gesture workaround cannot run on the first open (`<connections>` is under `*ngIf="open"` and `open` is set last), so it runs on the second open and calls `ngOnInit()` on an already initialized component, adding a second `getConnection()` subscription (released on destroy). |
| 11 | Low | `web-serial.ts` `poll()` | `value.buffer` ignores the chunk's `byteOffset`/`byteLength`; use `value` directly. Works today only because Chrome hands out chunks that own their buffer. |
| 12 | Low | `app.module.ts` `provideTranslateService()` | `lang: "de"` is hard-coded, so `de.json` is fetched on every start and templates render in German until `AppComponent.setLanguage()` switches. Use `'en'` or omit. |
| 13 | Low | `demo.ts` `DemoBackend.scan()` | Every `scan()` builds a new `DemoPeripheral` whose constructor starts eight `Car` timer loops that are never stopped and push into `laps` unbounded; each menu open or `AppComponent.connect()` adds another set. Demo mode only. |
| 14 | Low | `session.ts` `ranking`; `rms.page.ts` `startSession()` | `ranking` is not shared and `RmsPage` subscribes it four times, so each `reset` emission queues `cu.setMask()` four times (each displacing a poll cycle) and the `scan` state is kept four times over. `shareReplay({ bufferSize: 1, refCount: true })` on `ranking` fixes both. |
| 15 | Low | `firmware.page.ts` | `private requests = []` is `any[]` (type it `DataView[]`); `confirmFirmwareUpdate()` is `async` without awaiting `alert.show()`; `ngOnInit()` is an empty placeholder. |
| 16 | Info | `data-view.ts` `from()`; `control-unit.ts` `setLap()` | `0x30 \| value` assumes nibble values; `setLapHi(value >> 4)` passes 16+ for laps ≥ 256, which folds to `'0'`, so the CU lap display wraps at 255. Worth clamping or documenting. |
| 17 | Info | `app.component.ts` `connect()` | When the 3 s scan for a newly selected device times out, the previous `ControlUnit` stays active, and overlapping `getConnection()` emissions overwrite `stateSubscription` without unsubscribing. Small, accepted. |
| 18 | Info | `session.ts` `createGrid()` `scan()` | `this.finish(id)` inside the accumulator is a side effect in timing-critical code; the `finished` tuple flag would allow a downstream `tap()` on the `false → true` edge. Stays until it can be moved and tested on hardware (*Maintainer Decisions*). |
| 19 | Info | `control-unit.ts` `data` / `status` / `getState()` | `resetOnDisconnect: false` and `shareReplay({ refCount: false })` are deliberate so late subscribers keep working across reconnects (*Maintainer Decisions*). |

No other implementation issues found; `backendFilter` is guarded at every consumer
(`rms.page.ts`, `connections.component.ts`, `connection.page.ts`), and component
subscriptions under `settings/`, `drivers/`, `tuning/` and `menu/` are stored and
released in `ngOnDestroy`.

## Build & CI

| # | Priority | Location | Finding |
|---|----------|----------|---------|
| 1 | Low | `.github/workflows/ci.yml` | `actions/checkout@v3`, `actions/setup-node@v3` and `actions/setup-java@v2` run on deprecated Node 16 runtimes; bump to v4/v5. |
| 2 | Low | `.github/dependabot.yml` | `target-branch: "develop"` names a branch that does not exist on the remote, so action updates are never opened. |
| 3 | Info | `angular.json` | Declares `test` (karma) and `lint` (angular-eslint) targets whose dependencies are not installed; harmless, but misleading. |

## Translations (`src/assets/i18n/`)

No issues found. All nine files share the same 116 keys; placeholders and product
names are preserved.

## Docs (`README.md`)

No issues found. Build instructions, supported firmware versions and store links
are current.

## `CHANGELOG.md`

No issues found. The 2.7.2 section (2026-08-07) matches `config.xml` and the
format is consistent (imperative one-liners, two blank lines between releases).

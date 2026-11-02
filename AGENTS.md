# AGENTS.md — Open Lap

**Open Lap** is a slot car race management app for Carrera® DIGITAL 124/132 systems.
It talks to the track's Control Unit (CU) over Bluetooth LE (Carrera Bluetooth
Connect / AppConnect), Web Bluetooth or Web Serial, and ships as a PWA, an Android
app and an iOS app. Angular 20 (NgModule-based), Ionic 8, Apache Cordova, RxJS 7,
TypeScript 5.8. Single maintainer; `master` is the release branch and deploys the
PWA to GitHub Pages. Guidance here applies to every coding agent (Copilot, Claude
Code, Codex, …); keep it accurate when you change a convention.

## Ground Rules

- **The only check is `npm run build`** (strict templates; template errors fail it).
  There is no test suite, linter or formatter: `angular.json` still declares `test`
  and `lint` targets, but karma/eslint are not installed and there are no
  `*.spec.ts` files. Do not add tests, linters, formatters, state-management
  libraries or standalone components.
- **Do not edit `CHANGELOG.md` or bump the version** (`version` and
  `android-versionCode` in `config.xml`); both are release-only. Entries are
  `X.Y.Z YYYY-MM-DD` headings underlined with dashes, imperative one-line bullets
  ending with a period, blank lines between bullets, two blank lines between releases.
- **Hardware paths cannot be verified in CI** (`carrera/`, `backend/`,
  `rms/session.ts`, firmware update). Say so in your summary and describe what to
  test on a real CU (connect → drop link → reconnect; a multi-car race run to its
  lap/time limit; a full firmware update).
- Commit subjects are short imperative sentences ending with a period
  (`Modernize reconnect handling.`), one logical change per commit. Dependabot
  targets a `develop` branch; check which branches exist on the remote before
  choosing a base.
- Style: single quotes, semicolons, 2-space indentation; `strict: true` with
  `strictNullChecks: false`, `noImplicitAny: false`, `useDefineForClassFields: false`.
  Loose equality (`==`/`!=`) and legacy template syntax (`*ngIf`, `*ngFor`,
  `[ngSwitch]`) are the convention; do not convert them, not even as drive-bys.
  One `import … from 'rxjs'` and one `… from 'rxjs/operators'` per file. Existing
  `TODO`/`FIXME` comments record known gaps; leave them unless you fix the gap.
- Files: `kebab-case.component.ts` / `.page.ts` / `.service.ts` / `.module.ts` /
  `.pipe.ts` / `.directive.ts` / `.menu.ts` (popovers); domain models have no suffix
  (`control-unit.ts`). Templates in `.html` (`templateUrl`), styles in `.scss`
  (`styleUrls`, often omitted). Every feature directory has an `index.ts` barrel;
  import through it (`'../services'`, `'../carrera'`), not deep paths.
- Pages are routed and have no `selector`; components have a short unprefixed one
  (`checkerboard`, `gauge`). Popovers and modals receive data via `@Input()` passed
  as `componentProps` and dismiss through `PopoverController`/`ModalController`.
  Pipes carry both `@Pipe()` and `@Injectable()`.
- Translations (`src/assets/i18n/`, de en es fr hu it pl pt sk) must keep exactly
  the key set of `en.json`; keys are natural English strings with `{{placeholders}}`.
  Never translate product names (`Control Unit`, `Autonomous Car`, `Pace Car`,
  `Demo Control Unit`, `Carrera®`); prefer natural phrasing; match sentence-initial
  capitalization. French: `fr-FR`, "Aucun" for "None", "Il reste une minute !"
  (space before `!`/`?` is fine). Key-parity check:

  ```
  node -e 'const fs=require("fs"),d="src/assets/i18n/",en=Object.keys(JSON.parse(fs.readFileSync(d+"en.json")));for(const f of fs.readdirSync(d)){const k=Object.keys(JSON.parse(fs.readFileSync(d+f)));console.log(f,en.filter(x=>!k.includes(x)),k.filter(x=>!en.includes(x)))}'
  ```

## Architecture

**Layout** (`src/app/`): `app.module.ts` (root module, Cordova plugin providers,
`LoggingErrorHandler`, service worker only for production non-Cordova builds);
`app-routing.module.ts` (hash routing, `PreloadAllModules`, default `rms/practice`,
only `SettingsModule` lazy-loaded with inline child routes); `app-settings.ts`
(`AppSettings` facade over `SettingsService`/`@ionic/storage-angular` plus
`Connection`, `Options`, `RaceOptions`, `Driver` types); `backend/`, `carrera/`,
`drivers/`, `menu/`, `rms/` (`RmsPage`, `Session`, `leaderboard/`, `race-control/`),
`services/`, `settings/` (incl. `firmware.page.ts`), `shared/` (pipes, directives,
re-exports `TranslateModule`); `src/theme/` (dark theme only).

**Angular layer**: services are `@Injectable({ providedIn: 'root' })`;
`ControlUnitService` extends `BehaviorSubject<ControlUnit>` and components pipe
from it directly. No state library: subscribe to `AppSettings` Observables in
`ngOnInit`, keep a `private subscription: Subscription`, unsubscribe in
`ngOnDestroy` (some components use `firstValueFrom()`). `OnPush` only on
leaf/presentational components (`Checkerboard`, `Gauge`, `race-control/`,
`leaderboard/`), never on pages. Template-driven `FormsModule` everywhere except
`ReactiveFormsModule` for `RaceSettingsComponent`. i18n is `@ngx-translate` v17
with functional providers; templates use `<span translate>` and `| translate`.

**Backends** (`backend/`): abstract `Backend` with `abstract scan(): Observable<Peripheral>`;
`BLEBackend`, `WebBluetoothBackend`, `WebSerialBackend`, `DemoBackend` are
`@Injectable()` without `providedIn`, registered as a multi-provider in
`BackendModule` and injected as `@Inject(Backend) backends: Backend[]`.
`Peripheral.connect()` returns a `Subject<ArrayBuffer>` built with `createSubject()`
(`carrera/peripheral.ts`): `next()` writes to the device, subscribing re-runs a
**cold** observable that performs the physical connect, so every `retry()`
resubscription re-establishes the link. Intentional. `Peripheral.fwuBlockSize` is
the max firmware chunk size: BLE peripherals and the demo set `18`, Web Serial
leaves it `undefined`.

**Domain model** (`carrera/`, plus `Session` in `rms/`): plain TypeScript, no
decorators, instantiated directly, so it cannot use `LoggingService`. The local
`DataView` shadows the browser's; watch imports. `ControlUnit.data` is
`connectable(…, { resetOnDisconnect: false })`; the default would create a fresh
subject on every reconnect and orphan the long-lived
`shareReplay({ refCount: false })` consumers (`status`, `getFuel`, `getStart`, …).
Reconnect uses `retry({ resetOnSuccess: true, delay })`: the callback ignores the
error (`_error`), sets `state` to `'disconnected'` and returns an exponential
`timer()` that sets `'connecting'`; failures surface through `getState()`, not logs.
Never reintroduce `publish()`/`retryWhen()`. `Connection.backendFilter` is
`undefined` for "auto": guard it (`backendFilter?.includes(...)`) before string
methods, or the observable errors permanently. `Session` drives lap/time logic with
`groupBy` + `scan`; the `this.finish(id)` call inside `scan()` is a known,
timing-critical side effect (TODO in source). Do not change CU protocol constants,
command framing or reconnect timings without a hardware-backed reason.

**Firmware update** (`settings/firmware.page.ts`, not `ControlUnit`): bypasses
polling and talks to `cu.peripheral.connect()` directly. Input is a `.hmf` file of
`"<hex>"` lines (~880; outside 800–1000 lines is rejected except with the demo
backend); other names are rejected. `startFirmwareUpdate()` disconnects the CU,
waits 5 s, opens a raw connection, sends `GB2` (`FWU_START_COMMAND`) and feeds each
text-decoded response to `update()`; `G` responses (CU busy) are delayed 2 s via
`delayWhen`. `update()` drains queued `requests`, else sends the next line as one
`E` frame (no `fwuBlockSize`) or as `F` chunks of `fwuBlockSize` bytes plus `E0`
(`FWU_FINALIZE_COMMAND`); with no lines left it sets `status = 'finishing'`, waits
5 s and sends `0` (`VERSION_COMMAND`); a `0…` response carries the new version,
sets `'done'` and reconnects via `AppComponent.menu.reconnect()`. `status` is
`'init' | 'starting' | 'updating' | 'finishing' | 'done'`;
`firmwareCanDeactivateGuard` blocks navigation while `busy`. Framing
(`DataView.fromHex(cmd, hex)`): `F` frames carry the block length in byte 1 and no
CRC; other frames append an ASCII CRC byte (low nibble of the summed hex digit
values). `DataView.from(cmd, …values)` is the CRC scheme for ordinary CU commands.

## Development and Build

```bash
npm ci                                  # Node >= 20.5
npm start                               # ng serve; use the "Demo" backend without hardware
npm run build                           # production bundle to www/; the required check
ionic build --prod -- --base-href=/openlap/ --service-worker   # CI's PWA build
ionic cordova build android --prod      # native builds need Android SDK / Xcode
ionic cordova build ios --prod
```

Web Bluetooth and Web Serial need Chrome or Edge in a secure context. CI
(`.github/workflows/ci.yml`) builds android, ios and pwa on every push and PR and
deploys the PWA from `master`. Never commit `platforms/`, `plugins/`, `www/` or
`build.json`.

## Code Review

- Reviews **amend `.github/copilot-review.md` in place** (header: date, commit,
  build status, tests, one *Verified* paragraph); never new files, dated sections or
  a per-commit change summary (git history is the record).
- Findings are tables ordered High, Medium, Low, Info with columns
  `# | Priority | Location | Finding`; renumber after changes; cross-reference as
  "Code #3". **Resolved issues are removed**, never marked resolved.
- Keep it brief: one or two sentences per row, numbers only where they decide
  something. Keep accepted trade-offs as Info rows; record settled matters under
  *Maintainer Decisions*. A review changes nothing; fix only when asked.
- Do not flag the conventions under *Ground Rules* (loose equality, legacy template
  syntax, NgModules, split `rxjs` imports, missing tests, `strictNullChecks: false`
  consequences unless they cause a bug). Do flag protocol/framing errors, reconnect
  semantics, RxJS side effects, subscription leaks, unguarded `backendFilter`,
  translation key drift, and unrelated reformatting or version bumps in a change.

## Maintainer Decisions (do not re-raise)

- Loose equality and legacy template syntax stay; they are conventions, not defects.
- No automated tests or lint; hardware behaviour is verified manually on a CU.
- `ControlUnit.data` keeps `resetOnDisconnect: false`; `status` and `getState()`
  keep `shareReplay({ bufferSize: 1, refCount: false })` so late subscribers get
  the last value.
- The reconnect `delay` callback deliberately does not log the error; state
  transitions are the signal.
- `Session.createGrid()` keeps an unbounded `shareReplay({ refCount: true })` so
  late subscribers replay lap history.
- `this.finish(id)` stays inside `Session`'s `scan()` until it can be moved and
  tested on hardware in one go.

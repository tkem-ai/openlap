# AGENTS.md — Open Lap

Guidance for AI coding agents (GitHub Copilot, Claude Code, Codex, …) working in
this repository. Humans are welcome too. Keep this file accurate: when you change
a convention described here, update it in the same change.

## Project overview

Open Lap is a slot car race management app for Carrera® DIGITAL 124/132 systems.
It talks to the track's Control Unit (CU) over Bluetooth LE (Carrera Bluetooth
Connect / AppConnect), Web Bluetooth or Web Serial, and ships as a Progressive Web
App, an Android app and an iOS app.

- Stack: Angular 20 (NgModule-based), Ionic 8, Apache Cordova, RxJS 7, TypeScript 5.8.
- License: Apache-2.0. Carrera® is a trademark of Carrera Toys GmbH; Open Lap is not
  affiliated with them.
- User-facing documentation lives in `README.md`; release notes in `CHANGELOG.md`.

## Repository layout

```
src/app/
  app.module.ts          root module, Cordova plugin providers, LoggingErrorHandler
  app-routing.module.ts  hash routing; only SettingsModule is lazy-loaded
  app-settings.ts        AppSettings facade + Connection/Options/RaceOptions/Driver types
  backend/               Backend implementations: ble, web-bluetooth, web-serial, demo
  carrera/               pure-TypeScript domain model (ControlUnit, DataView, Peripheral)
  drivers/, tuning/      pages
  menu/                  side menu (connect/reconnect, navigation)
  rms/                   race management: RmsPage, Session, leaderboard/, race-control/
  services/              injectable services (settings, control-unit, logging, speech, …)
  settings/              lazy-loaded settings pages, incl. firmware.page.ts
  shared/                SharedModule: pipes, directives, re-exports TranslateModule
src/assets/i18n/         translations (de, en, es, fr, hu, it, pl, pt, sk)
src/theme/               Ionic CSS variables (dark theme only)
config.xml               Cordova config; holds the app version and android-versionCode
.github/workflows/ci.yml builds android, ios and pwa; deploys the PWA from master
```

Every feature directory has an `index.ts` barrel. Import through the barrel
(`'../services'`, `'../carrera'`) rather than deep paths.

## Setup and commands

```
npm ci                     # install dependencies (Node >= 20.5)
npm start                  # ng serve, dev server in the browser (use the "Demo" backend)
npm run build              # ng build, production bundle to www/
ionic build --prod -- --base-href=/openlap/ --service-worker   # what CI builds for the PWA
ionic cordova build android --prod   # native builds (need Android SDK / Xcode)
ionic cordova build ios --prod
```

`ionic serve` works too once the Ionic CLI is installed. Web Bluetooth and Web
Serial need Chrome or Edge and a secure context; the Demo backend needs no hardware.

## Verifying your work

- There is **no test suite and no linter**. `angular.json` still declares `test`
  and `lint` targets, but karma/eslint are not installed and there are no
  `*.spec.ts` files, so `ng test` / `ng lint` do not run. Do not add test files or
  linter config unless explicitly asked.
- The minimum check before you finish is a clean `npm run build` (strict
  templates are on, so template errors fail the build).
- Hardware-related changes (`carrera/`, `backend/`, `rms/session.ts`, firmware
  update) cannot be verified in CI. Say so in your summary and describe what to
  test on a real Control Unit (connect → drop link → reconnect; a multi-car race
  run to its lap/time limit; etc.).
- When touching translations, check key parity (see Internationalization).

## Code style

- Single quotes, semicolons, 2-space indentation. No formatter is configured;
  match the surrounding code.
- TypeScript `strict: true` but `strictNullChecks: false` and `noImplicitAny: false`;
  `useDefineForClassFields: false`. Keep it that way.
- Loose equality (`==` / `!=`) is the established convention project-wide. It is
  not a defect; do not "fix" it, and do not flag it in reviews.
- Legacy Angular template syntax (`*ngIf`, `*ngFor`, `[ngSwitch]`), not `@if`/`@for`.
- NgModules everywhere; every component/pipe/directive declares `standalone: false`.
- Templates in `.html` files (`templateUrl`), styles in `.scss` (`styleUrls`, often omitted).
- One `import { … } from 'rxjs'` and one `… from 'rxjs/operators'` per file; do
  not split them into several lines.
- Existing `TODO`/`FIXME` comments document known gaps. Leave them unless you fix
  the underlying issue.

### Naming

| Item          | Pattern                     | Example                     |
| ------------- | --------------------------- | --------------------------- |
| Components    | `kebab-case.component.ts`   | `checkerboard.component.ts` |
| Pages         | `kebab-case.page.ts`        | `rms.page.ts`               |
| Services      | `kebab-case.service.ts`     | `settings.service.ts`       |
| Modules       | `kebab-case.module.ts`      | `rms.module.ts`             |
| Pipes         | `kebab-case.pipe.ts`        | `time.pipe.ts`              |
| Directives    | `kebab-case.directive.ts`   | `target.directive.ts`       |
| Popover menus | `kebab-case.menu.ts`        | `rms.menu.ts`               |
| Domain models | `kebab-case.ts` (no suffix) | `control-unit.ts`           |

- Pages are routed to and have no `selector`. Components have a short,
  unprefixed selector (`checkerboard`, `gauge`, `connections`).
- Popover menus and modal dialogs receive data via `@Input()` properties passed
  as `componentProps` and dismiss through `PopoverController` / `ModalController`.
- Pipes carry both `@Pipe()` and `@Injectable()`.

## Architecture notes

### Angular layer

- Services are `@Injectable({ providedIn: 'root' })`. `ControlUnitService`
  extends `BehaviorSubject<ControlUnit>`; components pipe from it directly.
- Settings: no state library. `SettingsService` wraps `@ionic/storage-angular`;
  `AppSettings` (`app-settings.ts`) exposes typed Observables. Subscribe in
  `ngOnInit`, keep a `private subscription: Subscription`, unsubscribe in
  `ngOnDestroy` (some components use `firstValueFrom()` instead).
- `ChangeDetectionStrategy.OnPush` only on leaf/presentational components
  (`Checkerboard`, `Gauge`, the `race-control/` and `leaderboard/` components),
  never on pages.
- Template-driven `FormsModule` almost everywhere; `ReactiveFormsModule` only
  in `RmsModule` for `RaceSettingsComponent`.
- Routing is hash-based with `PreloadAllModules`; default route is `rms/practice`;
  `SettingsModule` defines its child routes inline.
- Service worker is enabled only for production, non-Cordova builds.

### Backends

- Abstract `Backend` (`backend/backend.ts`) with `abstract scan(): Observable<Peripheral>`.
- Implementations (`BLEBackend`, `WebBluetoothBackend`, `WebSerialBackend`,
  `DemoBackend`) are `@Injectable()` without `providedIn` and registered as a
  multi-provider in `BackendModule`; consumers inject `@Inject(Backend) backends: Backend[]`.
- `Peripheral.connect()` returns a `Subject<ArrayBuffer>` built with
  `createSubject()` (`carrera/peripheral.ts`): `next()` writes to the device and
  subscribing re-runs a **cold** observable that performs the physical connect.
  Every `retry()` resubscription therefore re-establishes the link. This is
  intentional.
- `Peripheral.fwuBlockSize` (optional) is the max firmware chunk size. BLE-based
  peripherals and the demo set `18`; Web Serial leaves it `undefined`.

### Domain model (`src/app/carrera/`)

- Plain TypeScript, no Angular decorators. `ControlUnit` and `Session` (`rms/`)
  are instantiated directly, never injected, so they cannot use `LoggingService`.
- The local `DataView` class shadows the browser's `DataView`. Watch your imports.
- `ControlUnit.data` is built with `connectable(…, { resetOnDisconnect: false })`.
  Keep `resetOnDisconnect: false`: the default would create a fresh subject on
  every reconnect and silently orphan the long-lived
  `shareReplay({ refCount: false })` consumers (`status`, `getFuel`, `getStart`, …).
- Reconnect/backoff uses `retry({ resetOnSuccess: true, delay })`. The `delay`
  callback ignores the error (`_error`), flips `state` to `'disconnected'`, and
  returns an exponential `timer()` that flips it to `'connecting'`. Failures are
  observable through `getState()`, not through logs. Do not reintroduce the
  deprecated `publish()` / `retryWhen()`.
- `Connection.backendFilter` (`app-settings.ts`) is `undefined` for "auto"
  detection. Guard it (`backendFilter?.includes(...)` or `if (backendFilter)`)
  before calling string methods; an unguarded call inside `map()` errors the
  observable permanently.
- `Session` (`rms/session.ts`) drives lap/time logic with `groupBy` + `scan`. The
  `this.finish(id)` call inside `scan()` is a known, timing-critical side effect
  (see the TODO there); do not move it without testing on hardware.

### Firmware update (`settings/firmware.page.ts`)

Implemented in `FirmwarePage`, not in `ControlUnit`. It bypasses the normal
polling protocol and talks to `cu.peripheral.connect()` directly.

- Input: `.hmf` files, one `"<hex>"` line per record (~880 lines; files outside
  800–1000 lines are rejected unless the demo backend is active). Other file
  names are rejected.
- Flow: `confirmFirmwareUpdate()` → `startFirmwareUpdate()` disconnects the
  `ControlUnit`, waits 5 s, opens a raw connection, sends `GB2`
  (`FWU_START_COMMAND`) and feeds every text-decoded response into `update()`.
  Responses starting with `G` (CU busy, e.g. erasing) are delayed 2 s via `delayWhen`.
- `update()` is a small state machine: drain queued `requests` first; otherwise
  send the next line as a single `E` frame (no `fwuBlockSize`) or as `F` chunks
  of `fwuBlockSize` bytes followed by `E0` (`FWU_FINALIZE_COMMAND`); when no
  lines remain, set `status = 'finishing'`, wait 5 s and send `0`
  (`VERSION_COMMAND`). A response starting with `0` carries the new version,
  sets `status = 'done'` and reconnects via `AppComponent.menu.reconnect()`.
- `status` is `'init' | 'starting' | 'updating' | 'finishing' | 'done'`;
  `firmwareCanDeactivateGuard` blocks navigation while `busy`.
- Framing (`DataView.fromHex(cmd, hex)`, `carrera/data-view.ts`): `F` frames put
  the block length in byte 1 and carry no CRC; other frames append an ASCII CRC
  byte (low nibble of the summed hex digit values). `DataView.from(cmd, …values)`
  is the CRC scheme for ordinary CU commands.

## Internationalization

- `@ngx-translate/core` v17 with functional providers (`provideTranslateService()`,
  `provideTranslateHttpLoader()`); `TranslateModule` is re-exported from `SharedModule`.
- Keys are natural English strings (`"Connected to {{device}}"`), not dotted ids.
  Templates use both `<span translate>Text</span>` and `{{ 'Text' | translate }}`.
- All files in `src/assets/i18n/` must have exactly the same key set as `en.json`.
  When adding or removing a key, update every locale. Quick check:

  ```
  node -e 'const fs=require("fs"),d="src/assets/i18n/",en=Object.keys(JSON.parse(fs.readFileSync(d+"en.json")));for(const f of fs.readdirSync(d)){const k=Object.keys(JSON.parse(fs.readFileSync(d+f)));console.log(f,en.filter(x=>!k.includes(x)),k.filter(x=>!en.includes(x)))}'
  ```

- Translation rules: keep `{{placeholders}}` verbatim; never translate product
  names (`Control Unit`, `Autonomous Car`, `Pace Car`, `Demo Control Unit`,
  `Carrera®`); prefer natural phrasing over word-by-word; match sentence-initial
  capitalization. French: `fr-FR`, "Aucun" for "None", wording like
  "Il reste une minute !" (space before `!`/`?` is fine).

## Git, releases and pull requests

- Commit subjects are short imperative sentences ending with a period, e.g.
  `Modernize reconnect handling.` One logical change per commit.
- Pushes to `master` deploy the PWA to GitHub Pages via `ci.yml`. Dependabot is
  configured to target a `develop` branch; check which branches exist on the
  remote before choosing a base.
- Releases: add an entry at the top of `CHANGELOG.md` in the existing
  `X.Y.Z YYYY-MM-DD` format, and bump `version` and `android-versionCode` in
  `config.xml`. Do not bump versions as part of an unrelated change.
- In a PR description, state what was built and what still needs testing on
  real hardware.

## Things not to do

- Do not add tests, linters, formatters, state-management libraries, or
  standalone components.
- Do not convert `==` to `===`, legacy template syntax to control flow, or
  `rxjs/operators` imports to `rxjs` imports as drive-by changes.
- Do not change CU protocol constants, command framing or reconnect timings
  without a hardware-backed reason.
- Do not commit `platforms/`, `plugins/`, `www/` or `build.json` (see `.gitignore`).

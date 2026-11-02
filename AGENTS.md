# Open Lap

Slot car race management app for Carrera® DIGITAL 124/132. It talks to the track's
Control Unit (CU) over Bluetooth LE (Carrera Bluetooth Connect / AppConnect), Web
Bluetooth or Web Serial and ships as a PWA, an Android app and an iOS app. Angular 20
(NgModules), Ionic 8, Apache Cordova, RxJS 7, TypeScript 5.8. Single maintainer;
`master` is the release branch and deploys the PWA to GitHub Pages. Code reviews
follow `.github/copilot-instructions.md`.

## Commands

```bash
npm ci                                  # Node >= 20.5
npm start                               # ng serve; the "Demo" backend needs no hardware
npm run build                           # production bundle to www/; the only check
ionic build --prod -- --base-href=/openlap/ --service-worker   # CI's PWA build
ionic cordova build android --prod      # native builds need Android SDK / Xcode
ionic cordova build ios --prod
```

No test suite, linter or formatter exists; the `test` and `lint` targets in
`angular.json` have no installed dependencies. CI (`.github/workflows/ci.yml`) builds
android, ios and pwa on every push and PR. Web Bluetooth and Web Serial need Chrome
or Edge in a secure context.

## Code Style

- Single quotes, semicolons, 2-space indentation; match the surrounding code.
  `strict: true` with `strictNullChecks: false`, `noImplicitAny: false`,
  `useDefineForClassFields: false`.
- Loose equality (`==`/`!=`) and legacy template syntax (`*ngIf`, `*ngFor`,
  `[ngSwitch]`) are the convention; never convert them, not even as drive-bys.
- NgModules only, every declarable has `standalone: false`; templates in `.html`
  (`templateUrl`), styles in `.scss`. One `import … from 'rxjs'` and one
  `… from 'rxjs/operators'` per file.
- Files: `kebab-case.component.ts` / `.page.ts` / `.service.ts` / `.module.ts` /
  `.pipe.ts` / `.directive.ts` / `.menu.ts` (popovers); domain models have no suffix.
  Import through each directory's `index.ts` barrel (`'../services'`, `'../carrera'`).
- Pages are routed and have no `selector`; components have a short unprefixed one
  (`checkerboard`, `gauge`). Popovers and modals take `@Input()`s via `componentProps`
  and dismiss through `PopoverController`/`ModalController`. Pipes carry both
  `@Pipe()` and `@Injectable()`.
- Services are `@Injectable({ providedIn: 'root' })`. Subscribe to settings in
  `ngOnInit`, keep a `private subscription: Subscription`, unsubscribe in
  `ngOnDestroy` (or use `firstValueFrom()`). `OnPush` only on leaf components
  (`Checkerboard`, `Gauge`, `race-control/`, `leaderboard/`), never on pages.
- Existing `TODO`/`FIXME` comments record known gaps; leave them unless you fix the gap.

## Architecture Constraints

- `carrera/` is the pure-TypeScript domain model; `ControlUnit` and `Session` (`rms/`)
  are plain classes instantiated directly, never injected, so they cannot use
  `LoggingService`. The local `DataView` shadows the browser's; watch imports.
  `app-settings.ts` is the typed facade over `SettingsService`; no state library.
- Backends (`backend/`) extend abstract `Backend` (`scan(): Observable<Peripheral>`),
  are `@Injectable()` without `providedIn` and registered as a multi-provider in
  `BackendModule` (`@Inject(Backend) backends: Backend[]`).
- `Peripheral.connect()` returns a `Subject<ArrayBuffer>` from `createSubject()`:
  `next()` writes to the device, subscribing re-runs a cold observable that performs
  the physical connect, so every `retry()` resubscription re-establishes the link.
- `ControlUnit.data` is `connectable(…, { resetOnDisconnect: false })` and
  `status`/`getState()` use `shareReplay({ bufferSize: 1, refCount: false })`; keep
  both, or reconnects orphan long-lived subscribers. Reconnect is
  `retry({ resetOnSuccess: true, delay })` with exponential `timer()` backoff and
  `state` transitions `disconnected` → `connecting` → `connected`; the error itself
  is ignored. Never reintroduce `publish()`/`retryWhen()`.
- `Connection.backendFilter` is `undefined` for "auto"; guard it
  (`backendFilter?.includes(...)`) before string methods, or the observable errors
  permanently.
- `Session` drives lap/time logic with `groupBy` + `scan`; `createGrid()` keeps an
  unbounded `shareReplay` on purpose so late subscribers replay lap history.
- Firmware update lives in `FirmwarePage`, not `ControlUnit`, over a raw
  `cu.peripheral.connect()`: `GB2` starts; each `"<hex>"` line of a `.hmf` file goes
  out as one `E` frame or, with `Peripheral.fwuBlockSize` (18 on BLE and demo, unset
  on serial), as `F` chunks plus `E0`; `0` requests the version; `G` means busy.
  `DataView.fromHex()` frames them (`F`: length byte, no CRC; others: ASCII CRC
  nibble); `firmwareCanDeactivateGuard` blocks navigation while `busy`.
- Protocol constants, command framing and reconnect timings change only for a
  hardware-backed reason.

## Testing

- The required check is a clean `npm run build` (strict templates). Do not add tests,
  linters, formatters, state-management libraries or standalone components.
- Hardware paths (`carrera/`, `backend/`, `rms/session.ts`, firmware update) cannot
  be verified in CI: say so in your summary and describe what to test on a real CU
  (connect → drop link → reconnect; a multi-car race to its lap/time limit; a full
  firmware update).
- Translations (`src/assets/i18n/`: de en es fr hu it pl pt sk) must keep exactly the
  key set of `en.json`; check with
  `node -e 'const fs=require("fs"),d="src/assets/i18n/",en=Object.keys(JSON.parse(fs.readFileSync(d+"en.json")));for(const f of fs.readdirSync(d)){const k=Object.keys(JSON.parse(fs.readFileSync(d+f)));console.log(f,en.filter(x=>!k.includes(x)),k.filter(x=>!en.includes(x)))}'`.
  Keys are natural English strings with `{{placeholders}}`. Never translate product
  names (`Control Unit`, `Autonomous Car`, `Pace Car`, `Demo Control Unit`,
  `Carrera®`); prefer natural phrasing; match sentence-initial capitalization.
  French: `fr-FR`, "Aucun" for "None", "Il reste une minute !" (space before `!`/`?`).

## Boundaries

- Never edit `CHANGELOG.md` or bump `version`/`android-versionCode` in `config.xml`;
  both are release-only (`X.Y.Z YYYY-MM-DD` headings, imperative one-line bullets
  ending with a period, two blank lines between releases).
- Never commit `platforms/`, `plugins/`, `www/` or `build.json`.
- Do not re-raise the *Maintainer Decisions* in `.github/copilot-instructions.md`.

## Git

- Commit subjects are short imperative sentences ending with a period
  (`Modernize reconnect handling.`), one logical change per commit. Dependabot targets
  a `develop` branch; check which branches exist on the remote before choosing a base.
- Run `npm run build` before pushing. Update this file in the same PR that changes a
  convention, and keep it under 100 lines.

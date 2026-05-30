# rn-scaffold — Guide

Everything you need to know to use the plugin: what each skill does, what it sets up, the conventions it enforces, and the common pitfalls.

For a quick install/use reference, see the [README](../README.md). This doc is the deep dive.

---

## What the plugin does

`rn-scaffold` ships three Claude Code skills:

| Skill | Purpose |
|---|---|
| `/rn-setup` | Bootstraps a React Native CLI project. Asks six preference questions (package manager, state, API, navigation, env, CI), then sets up folder structure, path aliases, ESLint (incl. `no-console`), Prettier, Husky, TypeScript strict, Jest, a hardened `metro.config.js` that strips `console.log/info/debug` from production bundles, `.gitignore` secrets patterns, navigation, state, API layer, env config, and (opt-in) GitHub Actions for lint + tsc + test + weekly `yarn audit`. When env config is enabled, also wires Android `productFlavors` (development/stage/production) with a chosen bundle-ID strategy, a guided iOS scheme setup using `react-native-config`'s `BuildXCConfig.rb` pre-action pattern, a verifier script for iOS, and 18 env-aware npm scripts covering run / APK / AAB / clean across all three environments. |
| `/rn-core` | Run after `rn-setup`. Asks four more questions (brand colors, font, encrypted storage, optional utilities), then installs production dependencies (Reanimated v4, FlashList, MMKV v4, FastImage, GestureHandler, KeyboardController, BottomSheet, LinearGradient, SVG) and creates a typed theme system, the `useThemeStyles` hook, base components, the root `AppWrapper`, and the state stores. If encrypted storage was selected, also installs `react-native-keychain` + `react-native-get-random-values` (and `redux-persist` if `/rn-setup` chose Redux), generates `storage.ts` with a Keychain-bootstrapped random key, wires an async storage gate into `App.tsx`, and (for Redux) wraps the auth slice in `persistReducer` with a whitelist. The Q4 multi-select adds any subset of: a deep-linking skeleton (extractor + dynamic route resolver + `useDeepLink` cold/warm-start hook + `navigationRef` wiring), an error handler abstraction (dev-only console, prod-stub for future Sentry adoption), a Zustand-backed toast manager + Reanimated component, a `react-native-permissions`-backed location helper, and a `@react-native-community/netinfo`-backed network status hook + slim offline banner. |
| `/rn-security` | Run after `rn-core`. Generates the Android `network_security_config.xml` for release builds (TLS-only, system CAs only) plus a debug variant that allows cleartext only to Metro's localhost endpoints, and wires `android:networkSecurityConfig` into `AndroidManifest.xml`. Idempotent — re-running detects prior installation and skips. iOS App Transport Security is already on by default; no changes there. |

The skills are generic — no project-specific code. Brand colors, fonts, API layer, package manager, encryption, CI are all asked at scaffold time.

---

## When to use this vs. starting by hand

Use it for **every new React Native CLI project**. It saves ~30 minutes of repetitive setup and guarantees every project starts from the same foundation (same lint config, same folder structure, same provider stack), which makes context-switching between projects much cheaper.

If you're working in an **existing project that already has its own conventions**, don't run the skills — they assume a fresh template and will fight existing setup.

---

## What gets created

### After `/rn-setup`

```
src/
  assets/ { fonts/, images/, svgs/ }
  components/
  constants/    → env.ts (if env config chosen)
  hooks/
  navigation/   → RootNavigator.tsx, types.ts (if RN v7 chosen)
  screens/      → Home/HomeScreen.tsx
  services/     → axiosClient.ts (REST) or apolloClient.ts (GraphQL)
  store/        → store.ts + hooks.ts + slices/authSlice.ts (if Redux chosen)
                  (Zustand stores are deferred to /rn-core — see note below)
  theme/
  types/
  utils/
```

Plus:
- `.eslintrc.js` with `import/order` enforcement + `no-console` guard (allows `warn`/`error`)
- `metro.config.js` with `pure_funcs` to strip `console.log/info/debug` from production bundles
- `.prettierrc.js`, `.editorconfig`, `.nvmrc` (Node 22)
- `.gitignore` with `*.keystore` (except `debug.keystore`), `*.mobileprovision`, `*.p12`, plus `.env.*` if env config chosen
- `babel.config.js` with `module-resolver` for `@/*` aliases
- `tsconfig.json` with `strict: true` and `paths`
- `jest.config.js` with `moduleNameMapper` for `@/*` and `passWithNoTests: true`
- `.husky/pre-commit` running `lint-staged` (eslint + prettier on staged files)
- `App.tsx` wired with the chosen providers in the correct order
- `.env.development/.stage/.production` + `.env.example` if env config chosen (each carries `API_BASE_URL` and `WEB_BASE_URL`)
- Android `productFlavors { development; stage; production }` in `android/app/build.gradle` with `dotenv.gradle` + `envConfigFiles` mapping (env config only)
- iOS multi-env: `scripts/verify-ios-env.sh` + a one-time manual Xcode scheme setup guide using `react-native-config`'s `BuildXCConfig.rb` pre-action pattern (env config only — schemes are duplicated by the user in Xcode UI, not the plugin)
- 18 env-aware npm scripts (`yarn android:development`, `yarn ios:production`, `yarn apk:stage`, `yarn aab:development`, `yarn clean:metro`, etc.) when env config is on; baseline `yarn start`, `yarn start:reset-cache` always
- `.github/workflows/lint-and-typecheck.yml` + `audit.yml` (if Q6 = Yes)

> **Why no Zustand `storage.ts` / `authStore.ts` here:** they're written by `/rn-core` so the encryption choice (Q3 in `/rn-core`) can govern what the templates look like. Keeping the templates in one skill avoids two skills racing on the same files.

### After `/rn-core`

```
src/
  theme/
    resources/ { LightThemeResources.ts, DarkThemeResources.ts, index.ts }
    typography.ts
    spacing.ts
    FlexBox.ts
  store/
    storage.ts                → encrypted MMKV + Keychain bootstrap (if Q3 = Yes)
                                 or simple MMKV (if Q3 = No)
    authStore.ts              → Zustand with skipHydration (if Zustand + Q3 = Yes)
    store.ts                  → upgraded with redux-persist + whitelist (if Redux + Q3 = Yes)
    themeStore.ts             → useThemeState with OS dark-mode listener
  hooks/useThemeStyles.ts     → useThemeStyles, useThemeValues
  types/styleTypes.ts         → NamedStyles<T>
  components/
    skeleton/BaseSkeleton.tsx
    core/image/FastImage.tsx
    list/BaseList.tsx
    wrappers/ { ScreenWrapper.tsx, KeyboardAwareSVWrapper.tsx }
    wrapper/AppWrapper.tsx    → GestureHandler → SafeArea → BottomSheetModal
```

Plus, if Q3 = Yes:
- `index.js` updated to import `react-native-get-random-values` as the very first statement
- `App.tsx` updated to await `bootstrapStorage()` before rendering (and wrap Redux children in `<PersistGate>`)

Plus, if Q4 picked any of these (each independent):

- **Option 1 (Deep-link)**: `src/utils/deepLink/{extractPath, extractParams}.ts`, `src/navigation/{navigationRef, routesConfig, resolveRoute, navigateByPath}.ts`, `src/hooks/useDeepLink.ts`. `routesConfig.ts` ships empty (with a commented example) — you fill in the route patterns; everything else is generic infrastructure. Dynamic-only matcher: patterns with `:slug` extract path params; literal segments must match. `extractPath` strips `Config.WEB_BASE_URL` automatically, so no host config is needed once `.env.*` is filled in.
- **Option 2 (Error handler)**: `src/utils/errorHandler.ts` — dev console / prod stub. No external deps; swap the stub for Sentry/Crashlytics later without changing call sites.
- **Option 3 (Toast)**: `src/store/toastStore.ts` + `src/components/core/Toast.tsx` (Reanimated-based, dual position). Mounted in `AppWrapper`.
- **Option 4 (Location)**: installs `react-native-permissions`. `src/hooks/useLocationPermission.ts`. Reminders printed about adding the iOS `Info.plist` keys + Android manifest permission.
- **Option 5 (Network)**: installs `@react-native-community/netinfo`. `src/hooks/useNetworkStatus.ts` + `src/components/core/NetworkStatusBanner.tsx`. Banner mounted in `AppWrapper`; hook also exported.

`App.tsx` is updated to mount `<KeyboardProvider>` and `<AppWrapper>` around the navigator. The example `HomeScreen` is rewritten to demonstrate the recommended theme pattern.

### After `/rn-security`

```
android/app/src/
  main/res/xml/network_security_config.xml     → release: TLS-only, system CAs
  debug/res/xml/network_security_config.xml    → debug: localhost/127.0.0.1/10.0.2.2/10.0.3.2 cleartext for Metro
  main/AndroidManifest.xml                     → android:networkSecurityConfig wired into <application>
```

No iOS changes — App Transport Security is already on by default in the RN template's `Info.plist`.

---

## Install and use

### One-time setup (per machine)

```
/plugin marketplace add JasoliyaShruti/rn-scaffold-plugin
/plugin install rn-scaffold@rn-scaffold-marketplace
```

Then **fully quit and reopen Claude Code**. Skills load at session start.

Verify by typing `/` in any session — both `/rn-setup` and `/rn-core` should appear.

### In a new project

```
/rn-setup
```

Answer six questions in one message (e.g. `1, 1, 2, 1, 1, 1`):

1. Package manager — Yarn Classic (1), Bun (2), npm (3). Yarn Classic is the team default.
2. State management — Zustand (1), Redux Toolkit (2), None (3).
3. API layer — REST + Axios/React Query (1), GraphQL + Apollo (2), None (3).
4. Navigation — React Navigation v7 (1), None (2).
5. Environment config — Yes (1), No (2).
6. GitHub Actions CI — Yes (1), No (2). Generates `.github/workflows/lint-and-typecheck.yml` (PR/main) and `audit.yml` (weekly).

Then:

```
/rn-core
```

Answers four more questions:

1. Brand colors — hex codes or `skip`.
2. Font family — name or `skip`.
3. Encrypted local storage — Yes (recommended) or No. Yes uses a Keychain-stored random key to encrypt MMKV; No leaves storage in plaintext (prototype mode only).
4. Optional utilities — multi-select like `1,3,5` or `none`. Pick any subset of: deep-linking base / error handler / toast manager / location permission / network status + offline banner.

If you provided a font name, drop the corresponding `.ttf` files into `src/assets/fonts/` and run `npx react-native-asset` to link them.

Then add Android network hardening:

```
/rn-security
```

No questions — writes the Android network security config and wires it into `AndroidManifest.xml`. Idempotent, so re-running on an already-configured project is safe.

Then build:

```
cd ios && bundle exec pod install && cd ..  # required if /rn-core installed Keychain
yarn ios     # or yarn android
```

### Update to a new version

```
/plugin marketplace update rn-scaffold-marketplace
/plugin update rn-scaffold
```

Restart Claude Code.

### Uninstall

```
/plugin uninstall rn-scaffold
/plugin marketplace remove rn-scaffold-marketplace
```

---

## Conventions baked into the scaffold

Please follow these — they're enforced by ESLint where possible, and code review where not:

- **All imports use `@/` aliases.** Never `../../../` outside the same feature folder.
- **Every folder has a barrel `index.ts`.** Import via the barrel, not the file path.
- **Screens are UI only.** No API calls, no business logic, no Zustand writes — those go in hooks.
- **Themed styles via `useThemeStyles`.** Never hardcode colors. Static styles that don't depend on theme (e.g. `flex: 1`, absolute-fill shells, briefly-visible placeholders) can use `StyleSheet.create`. **Never write inline `style={{ ... }}` literals in JSX** — breaks referential equality and trips `no-inline-styles`.
- **FlashList for all lists.** Always set `estimatedItemSize`, use stable string keys (NOT array index), wrap items in `React.memo`.
- **No anonymous functions in JSX props.** `onPress={handlePress}`, not `onPress={() => doSomething()}`. Hoist `useCallback` handlers out of the JSX — calling `useCallback` inline in a prop creates new refs every render.
- **TypeScript strict is on.** No `any`. Don't suppress with `// @ts-ignore` without an explanation.
- **Auth tokens via Keychain-encrypted MMKV.** Run `/rn-core` with Q3 = Yes for any production build — that wires a randomly-generated key stored in the iOS Keychain / Android Keystore so values can't be read from disk. Never AsyncStorage. Plain MMKV (no Keychain key) is for prototypes only.
- **Animations via Reanimated v4** with the `react-native-worklets/plugin` babel plugin (NOT `react-native-reanimated/plugin`, which was v3).
- **`<KeyboardProvider>` must mount above any `KeyboardAwareScrollView`.** The scaffolded `App.tsx` already does this — don't strip it when refactoring.
- **Pre-commit hooks are mandatory.** Don't skip with `--no-verify`.
- **Every hook in `src/hooks/` and every util in `src/utils/` must have a unit test.** Screens get at least a snapshot.
- **Mock at the boundary** (Apollo, native modules, network) — never mock internal modules.
- **Every scaffold run must end with `lint --fix` + `tsc --noEmit` + `yarn test` all exiting 0.** The skills run these automatically; treat any failure as a blocker.

---

## Roadmap — deferred skills

These four are tracked for future development; none are implemented in v0.1.5.

| Skill | Scope |
|---|---|
| `/rn-firebase` | Firebase Analytics + Crashlytics + Performance + Remote Config + Sentry, bundled as one adoption. Adds reportToProvider wiring into the `errorHandler.ts` stub generated by `/rn-core` Q4. |
| `/rn-update-ota` | Force-update gate driven by Remote Config (min/latest version + maintenance flag), four variants (maintenance, force, dismissible stable, OTA-triggered), `compareVersions` semver utility, `AppUpdateBottomSheet` UI. OTA provider to be picked at skill build-time (CodePush was deprecated by Microsoft in 2025; RevoPush is the paid community fork; Expo OTA requires Expo). |
| `/rn-push` | FCM permission (iOS auth-status + Android 13+ POST_NOTIFICATIONS), token fetch + refresh, foreground/background/cold-start listeners, push-payload deep-link routing (hooks into `navigateByPath` from `/rn-core` Q4), and a vendor adapter — user picks WebEngage / MoEngage / CleverTap and the skill scaffolds that integration with a shared `EngagementProvider` interface so the same hook code works regardless of vendor. |
| `/rn-ci-cd` | Fastlane lanes (`beta`, `release`), GitHub Actions release workflow with environment dispatch, keystore-from-base64 secret pattern, AAB upload to Play Console, source-map upload to Sentry / Crashlytics. Separate from the basic lint+tsc+test workflows generated by `/rn-setup` Q6. |

## Upstream versions targeted

The skills are written against current majors. Verified compatible with:

| Library | Version |
|---|---|
| Node | 22.x |
| Yarn Classic | 1.22.x |
| React Native CLI | 0.80+ (tested on 0.85) |
| MMKV | v4 (uses `createMMKV` factory) |
| Apollo Client | v4 (imports from `@apollo/client/react`, requires `rxjs` peer dep) |
| Reanimated | v4 (babel plugin `react-native-worklets/plugin`) |
| safe-area-context | v5+ (`Edge[]` is readonly) |

Each snippet in the skills carries a `> **Why ...**` callout explaining the API choice. If a library ships another breaking major, those callouts are the place to start when adjusting.

---

## Troubleshooting

**`/rn-setup` doesn't appear after install.**
Fully quit and reopen Claude Code. Skills load at session start.

**Marketplace add fails with "repository not found".**
Confirm `github.com/JasoliyaShruti/rn-scaffold-plugin` opens in a browser. The repo is public.

**Install fails with "invalid manifest".**
File an issue with the error message — the manifest format occasionally changes between Claude Code versions.

**`yarn ios` fails with a Reanimated build error.**
Most likely the wrong babel plugin. Open `babel.config.js` and confirm the last plugin entry is `'react-native-worklets/plugin'`, NOT `'react-native-reanimated/plugin'`. The latter was v3 and was removed in v4.

**`npx tsc --noEmit` fails right after the scaffold.**
Almost always an upstream API drift. Open the failing file, find the corresponding `> **Why ...**` callout in the relevant skill, and adjust against the current library version. File an issue if the fix is non-obvious.

**Apollo project but `AppWrapper.tsx` doesn't include `QueryClientProvider`.**
Intentional — `@tanstack/react-query` isn't installed for GraphQL projects since Apollo owns the cache. If you legitimately need both side-by-side (REST endpoints alongside GraphQL), `yarn add @tanstack/react-query` and re-add the provider.

**`yarn test` errors with `Cannot find module '@/...'`.**
`jest.config.js` should contain a `moduleNameMapper` entry resolving `@/*` to `src/*`:
```js
moduleNameMapper: { '^@/(.*)$': '<rootDir>/src/$1' },
```

**`yarn install` fails with `"engine ... incompatible"` on `lint-staged`.**
Your `lint-staged` is `^17` (requires Node ≥22.22.1) but your Node is below that. Either upgrade Node (`nvm install 22 && nvm use 22`) or pin to v16 (`yarn add -D lint-staged@^16`).

**`/rn-core` ran but fonts aren't showing.**
Did you drop the `.ttf` files into `src/assets/fonts/` and run `npx react-native-asset`? The plugin can't include font files for licensing reasons — you add them yourself.

**Encrypted storage chosen but `crypto.getRandomValues is not a function` at launch.**
The `react-native-get-random-values` polyfill isn't loaded early enough. Open `index.js` and confirm `import 'react-native-get-random-values';` is the very first line (before `import { AppRegistry } from 'react-native';`). If something else touches `crypto` during eager evaluation, the polyfill MUST be in front of it.

**Encrypted storage chosen but the app shows a blank screen on first launch.**
That's the bootstrap gate doing its job — `App.tsx` is waiting for `bootstrapStorage()` to read the Keychain. Typical resolution time is 50–150 ms. If it stays blank for seconds, you likely have a missing pod install (`cd ios && bundle exec pod install`) or a Keychain permission issue. Replace the `return null` in `App.tsx` with a quick splash component while you debug.

**`/rn-security` ran but `gradlew assembleRelease` fails parsing the manifest.**
You probably have a manual `<network-security-config>` block already in `AndroidManifest.xml` that conflicts with the attribute the skill added. Run `git diff android/app/src/main/AndroidManifest.xml` to see what changed and reconcile by hand.

**`yarn audit` is failing CI on a transitive advisory I can't fix.**
That's the documented tradeoff (see `/rn-setup` Step 2.13). Switch the audit job to `continue-on-error: true` and triage advisories via GitHub issues weekly. Don't silently delete the job.

**`yarn ios:development` builds but env vars come back undefined.**
The Xcode scheme pre-action didn't run. Most common cause: you duplicated the scheme but didn't paste the `BuildXCConfig.rb` shell into Build → Pre-actions. Run `yarn verify:ios-env` — the script reports which scheme is missing the pre-action.

**`yarn android:stage` fails with `Variant 'stageDebug' does not exist`.**
The `productFlavors` block isn't in `android/app/build.gradle` (or the `dimension "env"` line is missing). Open the gradle file and confirm the flavours block matches `/rn-setup` §2.6.1. After editing, run `cd android && ./gradlew clean && cd ..` before re-running.

**Bundle-ID strategy hindsight — I picked Shared but now want Separate.**
Open `android/app/build.gradle` and add `applicationIdSuffix ".development"` / `".stage"` to the matching flavour. After changing, the dev install becomes a *different app* on the device — you'll need to uninstall the old one. iOS schemes already use `.env.*` for the bridging header, so no iOS change is needed.

**Deep-link skeleton is in place but tapping a link does nothing.**
Three things to check, in order: (1) `Config.WEB_BASE_URL` is set in the `.env.*` you're running — verify with `console.log(Config.WEB_BASE_URL)` (DEV only) somewhere on first render. (2) Your route is added to `src/navigation/routesConfig.ts` and the pattern matches — `/products/:handle` matches `https://yourdomain.com/products/aloe`, not `https://yourdomain.com/product/aloe` (singular). (3) Custom-scheme URLs (`myapp://…`) require an `<intent-filter>` in `AndroidManifest.xml` and `CFBundleURLTypes` in `Info.plist` — neither is added by the skill, so add them manually when you pick a scheme.

**Network banner shows even though I have internet.**
`isConnected` is `true` on captive-portal Wi-Fi (you're physically connected, but can't reach the internet). The banner ships keyed off `isConnected`. For real-reachability gating use `isInternetReachable` from the same `useNetworkStatus()` hook — it runs a reachability probe.

**Anything else.**
File an issue at github.com/JasoliyaShruti/rn-scaffold-plugin with the exact error, your plugin version, and your Claude Code version (`claude --version`).

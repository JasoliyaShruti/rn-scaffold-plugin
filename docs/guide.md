# rn-scaffold — Guide

Everything you need to know to use the plugin: what each skill does, what it sets up, the conventions it enforces, and the common pitfalls.

For a quick install/use reference, see the [README](../README.md). This doc is the deep dive.

---

## What the plugin does

`rn-scaffold` ships two Claude Code skills:

| Skill | Purpose |
|---|---|
| `/rn-setup` | Bootstraps a React Native CLI project. Asks five preference questions (package manager, state, API, navigation, env), then sets up folder structure, path aliases, ESLint/Prettier/Husky, TypeScript strict, Jest, navigation, state, API layer, env config — all wired up. |
| `/rn-core` | Run after `rn-setup`. Asks two more questions (brand colors, font), then installs production dependencies (Reanimated v4, FlashList, MMKV v4, FastImage, GestureHandler, KeyboardController, BottomSheet, LinearGradient, SVG) and creates a typed theme system, the `useThemeStyles` hook, base components, and the root `AppWrapper`. |

The skills are generic — no project-specific code. Brand colors, fonts, API layer, package manager are all asked at scaffold time.

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
  store/        → authStore.ts, storage.ts (if Zustand chosen)
  theme/
  types/
  utils/
```

Plus:
- `.eslintrc.js` with `import/order` enforcement
- `.prettierrc.js`, `.editorconfig`, `.nvmrc` (Node 22)
- `babel.config.js` with `module-resolver` for `@/*` aliases
- `tsconfig.json` with `strict: true` and `paths`
- `jest.config.js` with `moduleNameMapper` for `@/*` and `passWithNoTests: true`
- `.husky/pre-commit` running `lint-staged` (eslint + prettier on staged files)
- `App.tsx` wired with the chosen providers in the correct order
- `.env.dev/.stage/.prod` + `.env.example` if env config chosen

### After `/rn-core`

```
src/
  theme/
    resources/ { LightThemeResources.ts, DarkThemeResources.ts, index.ts }
    typography.ts
    spacing.ts
    FlexBox.ts
  store/themeStore.ts         → useThemeState with OS dark-mode listener
  hooks/useThemeStyles.ts     → useThemeStyles, useThemeValues
  types/styleTypes.ts         → NamedStyles<T>
  components/
    skeleton/BaseSkeleton.tsx
    core/image/FastImage.tsx
    list/BaseList.tsx
    wrappers/ { ScreenWrapper.tsx, KeyboardAwareSVWrapper.tsx }
    wrapper/AppWrapper.tsx    → GestureHandler → SafeArea → BottomSheetModal
```

`App.tsx` is updated to mount `<KeyboardProvider>` and `<AppWrapper>` around the navigator. The example `HomeScreen` is rewritten to demonstrate the recommended theme pattern.

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

Answer five questions in one message (e.g. `1, 1, 2, 1, 1`):

1. Package manager — Yarn Classic (1), Bun (2), npm (3). Yarn Classic is the team default.
2. State management — Zustand (1), Redux Toolkit (2), None (3).
3. API layer — REST + Axios/React Query (1), GraphQL + Apollo (2), None (3).
4. Navigation — React Navigation v7 (1), None (2).
5. Environment config — Yes (1), No (2).

Then:

```
/rn-core
```

Answers two more questions:

1. Brand colors — hex codes or `skip`.
2. Font family — name or `skip`.

If you provided a font name, drop the corresponding `.ttf` files into `src/assets/fonts/` and run `npx react-native-asset` to link them.

Then build:

```
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
- **Auth tokens via MMKV with encryption.** Never AsyncStorage, never plain MMKV.
- **Animations via Reanimated v4** with the `react-native-worklets/plugin` babel plugin (NOT `react-native-reanimated/plugin`, which was v3).
- **`<KeyboardProvider>` must mount above any `KeyboardAwareScrollView`.** The scaffolded `App.tsx` already does this — don't strip it when refactoring.
- **Pre-commit hooks are mandatory.** Don't skip with `--no-verify`.
- **Every hook in `src/hooks/` and every util in `src/utils/` must have a unit test.** Screens get at least a snapshot.
- **Mock at the boundary** (Apollo, native modules, network) — never mock internal modules.
- **Every scaffold run must end with `lint --fix` + `tsc --noEmit` + `yarn test` all exiting 0.** The skills run these automatically; treat any failure as a blocker.

---

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

**Anything else.**
File an issue at github.com/JasoliyaShruti/rn-scaffold-plugin with the exact error, your plugin version, and your Claude Code version (`claude --version`).

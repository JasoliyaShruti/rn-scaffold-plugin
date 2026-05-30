---
name: rn-core
description: Use after rn-setup to install production dependencies and create theme system, core components, hooks, and wrappers needed for PRD-driven UI generation
allowed-tools: Bash Read Write Edit Glob Grep
---

# React Native Core — Dependencies, Theme & Core Components

Install production dependencies and create the theme system, core hooks, core components, and root wrappers that every screen needs. Run this AFTER `rn-setup` has scaffolded the project.

**Prerequisite:** `rn-setup` completed (folder structure, path aliases, ESLint, navigation, state, API layer exist).

---

## Step 0 — Ask Preferences

Ask the user these four questions before any installs. They control the theme, typography, storage encryption, and optional utilities. Present them in a single message:

### Q1. Brand Colors
"Do you have brand colors for this project? Provide hex codes (e.g. `primary: #FF6B00, secondary: #1B1B1B`) or reply `skip` to use a neutral black/white default palette."

- Parse any provided colors into named slots (`primary`, `secondary`, `accent`, etc.).
- If `skip` or empty, use the default palette below.

### Q2. Font Family
"Which font family does this project use? Provide a name (e.g. `Inter`, `Roboto`) or reply `skip` to use the system default font."

- If a name is provided, set the `fontFamily` constants to `<Name>-Regular`, `<Name>-Medium`, `<Name>-SemiBold`, `<Name>-Bold`. After setup, tell the user to drop the corresponding `.ttf` files into `src/assets/fonts/` and run `npx react-native-asset` (or set up `react-native.config.js`) to link them.
- If `skip`, set every `fontFamily` value to `undefined` so React Native uses the system font. Do NOT install any font package.

### Q3. Encrypted Local Storage (recommended)
"Encrypt local storage with a randomly-generated key stored in the device Keychain (iOS) / Keystore (Android)? [Y/n — recommended for production]"

- Default is **Yes**. Reply `n` only if this is a throwaway prototype.
- **What it does:** generates a 32-byte random key on first launch, stores it in the OS Keychain/Keystore (`AFTER_FIRST_UNLOCK` accessibility), and uses it to encrypt MMKV at rest. The key never appears in your bundle or source. Adds an async bootstrap gate to `App.tsx` (renders blank for ~50–150ms on first launch while reading the Keychain).
- **Tradeoffs to flag to the user:**
  - **Key rotation is destructive.** Once values are encrypted with this key, you can't decrypt them with a different key — rotating means re-encrypting every value or wiping state on next launch. Plan a migration path before launch.
  - **Adds two deps:** `react-native-keychain`, `react-native-get-random-values`. Both maintained, both small. iOS Keychain APIs require a quick re-`pod install`.
  - **Doesn't protect from compromised devices** — a rooted/jailbroken device with the app's process can still read the decrypted in-memory copy. This is at-rest protection only, valuable for stolen-device and offline-backup scenarios.
- Skip applies if the user picked `None` for state management in `/rn-setup` — there's nothing to encrypt yet.

### Q4. Optional utilities (multi-select)

"Which optional utilities to include? Reply with the numbers you want — e.g. `1,3,5` or `none`:

  1) **Deep-linking base** — generic skeleton for handling deep links: URL extractor, dynamic-pattern route resolver, `navigateByPath`, `useDeepLink` hook with cold/warm-start handling, `NavigationContainer.onReady` wiring. Strips host using `Config.WEB_BASE_URL` from your `.env` files. **You fill in the actual routes; the skeleton handles arrival + matching + navigation.** Native manifest config (custom URL scheme, universal links) is NOT included — add later when you have a scheme + verified domain.

  2) **Error handler** — `src/utils/errorHandler.ts` with `__DEV__`-aware console behaviour (verbose in dev, silenced in prod). No external deps. Designed to be swappable later for Sentry/Crashlytics when you add `/rn-firebase`.

  3) **Toast manager** — `src/store/toastStore.ts` + `src/components/core/Toast.tsx`. Zustand-backed, dual position (top/bottom), auto-dismiss with cleanup, Reanimated-based show/hide. Mounted by `AppWrapper` so it's available everywhere.

  4) **Location permission helper** — `src/hooks/useLocationPermission.ts` with `check()` + `request()` + Android GPS-enable prompt. Installs `react-native-permissions` (needs `pod install`).

  5) **Network status** — `src/hooks/useNetworkStatus.ts` + `src/components/core/NetworkStatusBanner.tsx` (slim non-blocking banner at top of screen when offline, auto-hides on reconnect). Installs `@react-native-community/netinfo`. Banner wired into `AppWrapper`; hook also exported for screen-level use."

- Default if user replies empty: **`none`**.
- Each item is independent — pick any subset.
- Files for un-selected items are NOT generated and their deps are NOT installed.

Wait for all four answers before proceeding.

---

## Step 1 — Install Dependencies

Install all packages using the project's package manager (check `package.json` for yarn/npm/bun).

> **Conditional install — React Query:** `@tanstack/react-query` is required only when `rn-setup` selected REST + React Query (Q3 = 1). For Apollo (GraphQL) or `None`, skip it — see Step 7. The unconditional-install pattern from earlier versions of this skill shipped an unused package on Apollo projects.

### Animation & Gesture

```bash
yarn add react-native-reanimated react-native-worklets react-native-gesture-handler
```

> `react-native-worklets` MUST be installed as a direct dependency. Reanimated v4 extracted worklets into this separate package and it is a peer dep — Yarn Classic does not auto-install peer deps, and even on package managers that do, relying on auto-resolution is fragile (monorepo hoisting, etc.). The babel plugin imports from `react-native-worklets/plugin`, so the package has to be resolvable from `node_modules`.

**Babel config** — add to `babel.config.js` plugins (worklets plugin MUST be last):

```js
plugins: [
  // ... existing plugins (module-resolver, etc.)
  'react-native-worklets/plugin', // MUST be last
]
```

> Reanimated v4 uses worklets directly — use `react-native-worklets/plugin`, NOT the old `react-native-reanimated/plugin` (which was for v3 and is removed in v4).

### UI Components

```bash
yarn add @shopify/flash-list @gorhom/bottom-sheet @d11/react-native-fast-image react-native-linear-gradient react-native-svg
```

### Storage

```bash
yarn add react-native-mmkv react-native-nitro-modules
```

> `react-native-nitro-modules` is a required peer dependency of `react-native-mmkv`. Always install it alongside mmkv.

**If Q3 = Yes (encrypted storage):** install Keychain + a crypto-secure RNG polyfill:

```bash
yarn add react-native-keychain react-native-get-random-values
```

> **Why these two:** `react-native-keychain` is the bridge to iOS Keychain and Android Keystore — that's where we stash the encryption key so it never lands in the JS bundle, source control, or backups. `react-native-get-random-values` polyfills `crypto.getRandomValues()` (which RN's JS engine doesn't ship by default), used to generate the 32-byte random key on first launch. Without the polyfill, `crypto.getRandomValues` is undefined and key generation throws at runtime.

**If `/rn-setup` picked Redux Toolkit AND Q3 = Yes:** also install `redux-persist` for the persistence wrapper used in Step 4:

```bash
yarn add redux-persist
```

> Redux + encrypted storage = persist `auth` selectively via `redux-persist` with the encrypted-MMKV adapter. The persistence is whitelisted (auth slice only) — see Step 4. If Q3 = No, skip this install; Redux stays memory-only and matches what `/rn-setup` scaffolded.

### Keyboard

```bash
yarn add react-native-keyboard-controller
```

### iOS Pods

```bash
cd ios && pod install && cd ..
```

---

## Step 2 — Theme System

### 2.1 Colors — `src/theme/resources/LightThemeResources.ts`

If the user provided brand colors in Step 0, fold them into `STATIC_COLORS` and reference them from `LightThemeColors`. If they replied `skip`, use the default palette below as-is.

```typescript
const STATIC_COLORS = {
  black: '#111111',
  white: '#FFFFFF',
  transparent: 'transparent',
  // If brand colors provided, add them here, e.g.:
  // brandPrimary: '#FF6B00',
  // brandSecondary: '#1B1B1B',
};

export type ThemeColorsType = {
  tint: string;
  primary: string;
  secondary: string;

  text: string;
  textSecondary: string;
  textDisabled: string;
  textInverse: string;

  background: string;
  backgroundSecondary: string;
  backgroundInverse: string;

  border: string;
  borderFocus: string;

  success: string;
  error: string;
  warning: string;
  info: string;

  overlay: string;
  transparent: string;

  skeletonShimmerStart: string;
  skeletonShimmerEnd: string;
};

export const LightThemeColors: ThemeColorsType = {
  tint: '#111111',           // → STATIC_COLORS.brandPrimary if brand colors provided
  primary: STATIC_COLORS.black,    // → STATIC_COLORS.brandPrimary
  secondary: STATIC_COLORS.white,  // → STATIC_COLORS.brandSecondary

  text: '#111111',
  textSecondary: '#666666',
  textDisabled: '#AAAAAA',
  textInverse: STATIC_COLORS.white,

  background: STATIC_COLORS.white,
  backgroundSecondary: '#F5F5F5',
  backgroundInverse: '#111111',

  border: '#E0E0E0',
  borderFocus: '#111111',

  success: '#2E7D32',
  error: '#C62828',
  warning: '#F57F17',
  info: '#0277BD',

  overlay: 'rgba(0, 0, 0, 0.5)',
  transparent: STATIC_COLORS.transparent,

  skeletonShimmerStart: '#EEEEEE',
  skeletonShimmerEnd: '#D2D2D2',
};
```

> **Why `ThemeColorsType` is an explicit interface, not `typeof LightThemeColors`:** with `typeof` + `as const` (or even without), TypeScript infers literal types for every value (`primary: '#111111'`). The Dark theme then has *different* literal types for the same keys (`primary: '#FFFFFF'`), so `LightThemeColors | DarkThemeColors` collapses to an unhelpful union that doesn't satisfy a single `ThemeColorsType`. Result: `useThemeStyles` typechecks fail because `ThemeResources[theme]` doesn't widen to a single colors shape. The explicit-interface form keeps both palettes typed as `string` for each slot, and the union is structurally identical.

Create `src/theme/resources/DarkThemeResources.ts` — same shape, concrete dark values, explicitly typed:

```typescript
import type { ThemeColorsType } from './LightThemeResources';

const STATIC_COLORS_DARK = {
  black: '#111111',
  white: '#FFFFFF',
  transparent: 'transparent',
  // Same brand colors as light if provided
};

export const DarkThemeColors: ThemeColorsType = {
  tint: '#FFFFFF',
  primary: STATIC_COLORS_DARK.white,
  secondary: STATIC_COLORS_DARK.black,

  text: '#FFFFFF',
  textSecondary: '#AAAAAA',
  textDisabled: '#666666',
  textInverse: '#111111',

  background: '#111111',
  backgroundSecondary: '#1F1F1F',
  backgroundInverse: STATIC_COLORS_DARK.white,

  border: '#333333',
  borderFocus: STATIC_COLORS_DARK.white,

  success: '#4CAF50',
  error: '#EF5350',
  warning: '#FFA726',
  info: '#42A5F5',

  overlay: 'rgba(0, 0, 0, 0.7)',
  transparent: STATIC_COLORS_DARK.transparent,

  skeletonShimmerStart: '#1F1F1F',
  skeletonShimmerEnd: '#2D2D2D',
};
```

Create `src/theme/resources/index.ts`:

```typescript
import { DarkThemeColors } from './DarkThemeResources';
import { LightThemeColors, ThemeColorsType } from './LightThemeResources';

export const ThemeResources: Record<'light' | 'dark', ThemeColorsType> = {
  light: LightThemeColors,
  dark: DarkThemeColors,
};

export type { ThemeColorsType } from './LightThemeResources';
```

> **No `as const` on `ThemeResources`:** the `Record<...>` annotation gives us the right type. `as const` would re-narrow back to literal-typed entries, defeating the purpose of the interface.

### 2.2 Typography — `src/theme/typography.ts`

If the user provided a font name (e.g. `Inter`) in Step 0:

```typescript
import { StyleSheet } from 'react-native';

export const fontFamily = {
  regular: 'Inter-Regular',
  medium: 'Inter-Medium',
  semiBold: 'Inter-SemiBold',
  bold: 'Inter-Bold',
} as const;

export const typography = StyleSheet.create({
  h1: { fontFamily: fontFamily.bold, fontSize: 32, lineHeight: 40 },
  h2: { fontFamily: fontFamily.bold, fontSize: 24, lineHeight: 32 },
  h3: { fontFamily: fontFamily.semiBold, fontSize: 20, lineHeight: 28 },
  h4: { fontFamily: fontFamily.semiBold, fontSize: 18, lineHeight: 24 },
  body: { fontFamily: fontFamily.regular, fontSize: 16, lineHeight: 24 },
  bodySmall: { fontFamily: fontFamily.regular, fontSize: 14, lineHeight: 20 },
  caption: { fontFamily: fontFamily.regular, fontSize: 12, lineHeight: 16 },
  label: { fontFamily: fontFamily.medium, fontSize: 14, lineHeight: 20 },
  button: { fontFamily: fontFamily.semiBold, fontSize: 16, lineHeight: 24 },
});
```

Tell the user to drop `Inter-Regular.ttf`, `Inter-Medium.ttf`, `Inter-SemiBold.ttf`, `Inter-Bold.ttf` (or equivalent for their font) into `src/assets/fonts/`, then link with `npx react-native-asset` (or configure `react-native.config.js`).

If the user replied `skip` to Q2, use the system font instead — every `fontFamily` value is `undefined`:

```typescript
import { StyleSheet, TextStyle } from 'react-native';

export const fontFamily = {
  regular: undefined,
  medium: undefined,
  semiBold: undefined,
  bold: undefined,
} as const;

const weight = (w: TextStyle['fontWeight']): TextStyle => ({ fontWeight: w });

export const typography = StyleSheet.create({
  h1: { ...weight('700'), fontSize: 32, lineHeight: 40 },
  h2: { ...weight('700'), fontSize: 24, lineHeight: 32 },
  h3: { ...weight('600'), fontSize: 20, lineHeight: 28 },
  h4: { ...weight('600'), fontSize: 18, lineHeight: 24 },
  body: { ...weight('400'), fontSize: 16, lineHeight: 24 },
  bodySmall: { ...weight('400'), fontSize: 14, lineHeight: 20 },
  caption: { ...weight('400'), fontSize: 12, lineHeight: 16 },
  label: { ...weight('500'), fontSize: 14, lineHeight: 20 },
  button: { ...weight('600'), fontSize: 16, lineHeight: 24 },
});
```

No font files needed, no linking required.

### 2.3 Spacing — `src/theme/spacing.ts`

```typescript
export const spacing = {
  xs: 4,
  sm: 8,
  md: 12,
  lg: 16,
  xl: 20,
  xxl: 24,
  xxxl: 32,
  huge: 40,
  massive: 48,
  nano: 6,
  smPlus: 10,
  mdPlus: 14,
  lgPlus: 18,
} as const;

export type SpacingKey = keyof typeof spacing;
```

### 2.4 FlexBox Utilities — `src/theme/FlexBox.ts`

```typescript
import { ViewStyle } from 'react-native';

export const FlexBoxStyles: Record<string, ViewStyle> = {
  row: { flexDirection: 'row' },
  rowCenter: { flexDirection: 'row', alignItems: 'center' },
  rowCenterSpaceBetween: { flexDirection: 'row', alignItems: 'center', justifyContent: 'space-between' },
  rowCenterSpaceAround: { flexDirection: 'row', alignItems: 'center', justifyContent: 'space-around' },
  rowEnd: { flexDirection: 'row', alignItems: 'flex-end' },
  center: { alignItems: 'center', justifyContent: 'center' },
  flex1: { flex: 1 },
  flex1Center: { flex: 1, alignItems: 'center', justifyContent: 'center' },
  alignCenter: { alignItems: 'center' },
  justifyCenter: { justifyContent: 'center' },
  selfCenter: { alignSelf: 'center' },
  wrap: { flexWrap: 'wrap' },
};
```

### 2.5 Theme Barrel — `src/theme/index.ts`

```typescript
export { ThemeResources } from './resources';
export type { ThemeColorsType } from './resources';
export { typography, fontFamily } from './typography';
export { spacing } from './spacing';
export type { SpacingKey } from './spacing';
export { FlexBoxStyles } from './FlexBox';
```

---

## Step 3 — Types

### `src/types/styleTypes.ts`

```typescript
import { ImageStyle, TextStyle, ViewStyle } from 'react-native';

export type NamedStyles<T> = {
  [P in keyof T]: ViewStyle | TextStyle | ImageStyle;
};
```

### `src/types/index.ts`

```typescript
export type { NamedStyles } from './styleTypes';
```

---

## Step 4 — Stores + Optional Utilities

This step writes the store files that `/rn-setup` deliberately deferred. Three sub-steps:

- **4.1** Storage primitive (`storage.ts`) — encrypted MMKV bootstrap if Q3 = Yes; simple MMKV otherwise.
- **4.2** Auth store — Zustand `authStore.ts` OR Redux `store.ts` upgrade with `redux-persist`, depending on what `/rn-setup` chose.
- **4.3** Theme store (`themeStore.ts`) — universal, Zustand-based, with OS dark-mode integration.

If `/rn-setup` Q2 was `None` (no state management), skip 4.1 and 4.2; do only 4.3.

---

### 4.1 Storage primitive — `src/store/storage.ts`

**Variant A — Q3 = Yes (encrypted, recommended for production):**

```typescript
import 'react-native-get-random-values';

import { createMMKV, MMKV } from 'react-native-mmkv';
import * as Keychain from 'react-native-keychain';

const KEYCHAIN_SERVICE = 'app-storage-encryption';

const generateRandomKey = (): string => {
  const bytes = new Uint8Array(32);
  crypto.getRandomValues(bytes);
  return Array.from(bytes)
    .map(b => b.toString(16).padStart(2, '0'))
    .join('');
};

const getOrCreateEncryptionKey = async (): Promise<string> => {
  const existing = await Keychain.getGenericPassword({ service: KEYCHAIN_SERVICE });
  if (existing && existing.password) {
    return existing.password;
  }
  const key = generateRandomKey();
  await Keychain.setGenericPassword('mmkv-key', key, {
    service: KEYCHAIN_SERVICE,
    accessible: Keychain.ACCESSIBLE.AFTER_FIRST_UNLOCK,
  });
  return key;
};

let mmkv: MMKV | null = null;

export const bootstrapStorage = async (): Promise<MMKV> => {
  if (mmkv) {
    return mmkv;
  }
  const key = await getOrCreateEncryptionKey();
  mmkv = createMMKV({ id: 'app-storage', encryptionKey: key });
  return mmkv;
};

export const getStorage = (): MMKV => {
  if (!mmkv) {
    throw new Error('Storage not bootstrapped. Await bootstrapStorage() before reading.');
  }
  return mmkv;
};
```

If `/rn-setup` chose **Zustand**, also export the adapter:

```typescript
import type { StateStorage } from 'zustand/middleware';

export const zustandMMKVStorage: StateStorage = {
  getItem: name => getStorage().getString(name) ?? null,
  setItem: (name, value) => getStorage().set(name, value),
  removeItem: name => {
    getStorage().remove(name);
  },
};
```

If `/rn-setup` chose **Redux**, also export the redux-persist adapter:

```typescript
import type { Storage } from 'redux-persist';

export const reduxPersistMMKVStorage: Storage = {
  setItem: (key, value) => {
    getStorage().set(key, value);
    return Promise.resolve(true);
  },
  getItem: key => Promise.resolve(getStorage().getString(key) ?? null),
  removeItem: key => {
    getStorage().remove(key);
    return Promise.resolve();
  },
};
```

**Also update `index.js`** (the RN entry file, not `App.tsx`) so the crypto polyfill loads before *anything* else:

```js
import 'react-native-get-random-values';   // MUST be the first import
import { AppRegistry } from 'react-native';

import App from './App';
import { name as appName } from './app.json';

AppRegistry.registerComponent(appName, () => App);
```

> **Why polyfill ordering matters:** `getOrCreateEncryptionKey` calls `crypto.getRandomValues`. RN's JS engine (Hermes / JSC) doesn't ship that API by default — without the polyfill loaded first, the call throws `ReferenceError: crypto is not defined` before the storage bootstrap can finish. The `App.tsx`-level import is too late if any other module touches `crypto` during eager evaluation.

**Variant B — Q3 = No (simple, prototype-only):**

```typescript
import { createMMKV } from 'react-native-mmkv';
import type { StateStorage } from 'zustand/middleware'; // Zustand path
// import type { Storage } from 'redux-persist';        // Redux path

export const mmkv = createMMKV({ id: 'app-storage' });

// Zustand adapter:
export const zustandMMKVStorage: StateStorage = {
  getItem: name => mmkv.getString(name) ?? null,
  setItem: (name, value) => mmkv.set(name, value),
  removeItem: name => {
    mmkv.remove(name);
  },
};
```

> **No encryption, no key.** Anyone with filesystem access to the unpacked app can read these values in plaintext. Acceptable for prototypes; **not** acceptable for shipping a customer-facing build. Re-run `/rn-core` with Q3 = Yes before launch.

> **MMKV v4 API:** the constructor is `createMMKV({...})` (returns a Nitro hybrid object), the delete method is `remove(key)`. If you're pinned to v2/v3, use the old `new MMKV({...})` + `delete(key)` API.

---

### 4.2 Auth store

**If `/rn-setup` chose Zustand — create `src/store/authStore.ts`:**

```typescript
import { create } from 'zustand';
import { createJSONStorage, persist } from 'zustand/middleware';

import { zustandMMKVStorage } from './storage';

interface User {
  id: string;
  name: string;
  email: string;
}

interface AuthState {
  token: string | null;
  user: User | null;
  isAuthenticated: boolean;
  setAuth: (token: string, user: User) => void;
  clearAuth: () => void;
}

export const useAuthStore = create<AuthState>()(
  persist(
    set => ({
      token: null,
      user: null,
      isAuthenticated: false,
      setAuth: (token, user) => set({ token, user, isAuthenticated: true }),
      clearAuth: () => set({ token: null, user: null, isAuthenticated: false }),
    }),
    {
      name: 'auth-storage',
      storage: createJSONStorage(() => zustandMMKVStorage),
      // Q3 = Yes only: storage is encrypted and bootstrapped asynchronously, so
      // we must defer hydration until App.tsx awaits bootstrapStorage().
      skipHydration: true,
    },
  ),
);
```

> If Q3 = No, omit `skipHydration: true` — the simple MMKV instance is ready synchronously, no gate needed.

**Persistence rule (both Zustand and Redux):** future stores that hold transient commerce data (cart, search results, query cache) **must not** use `persist`. Persist identity (`auth`, `userPreferences`); fetch the rest fresh on cold start. Stale prices and stale inventory are real UX problems.

**If `/rn-setup` chose Redux AND Q3 = Yes — overwrite `src/store/store.ts`:**

```typescript
import { combineReducers, configureStore } from '@reduxjs/toolkit';
import {
  FLUSH,
  PAUSE,
  PERSIST,
  persistReducer,
  persistStore,
  PURGE,
  REGISTER,
  REHYDRATE,
} from 'redux-persist';

import authReducer from './slices/authSlice';
import { reduxPersistMMKVStorage } from './storage';

const rootReducer = combineReducers({
  auth: authReducer,
  // Add new slices here. Only whitelist the ones that should survive restarts.
});

const persistedReducer = persistReducer(
  {
    key: 'root',
    storage: reduxPersistMMKVStorage,
    whitelist: ['auth'], // persist identity only — see "Persistence rule" above
  },
  rootReducer,
);

export const store = configureStore({
  reducer: persistedReducer,
  middleware: getDefaultMiddleware =>
    getDefaultMiddleware({
      serializableCheck: {
        // redux-persist dispatches non-serializable actions during rehydration;
        // ignore them or the default serializableCheck floods the console.
        ignoredActions: [FLUSH, REHYDRATE, PAUSE, PERSIST, PURGE, REGISTER],
      },
    }),
});

export const persistor = persistStore(store);

export type RootState = ReturnType<typeof store.getState>;
export type AppDispatch = typeof store.dispatch;
```

> **If Q3 = No:** leave `store.ts` exactly as `/rn-setup` wrote it (no `persistReducer`). Redux runs memory-only; nothing survives restarts. Skip the rest of this sub-step.

---

### 4.3 Theme store — `src/store/themeStore.ts`

> **Why this is always a Zustand store, even on Redux projects:** the theme is universal across every project — same shape, three states, OS listener. Re-implementing it as a Redux slice adds a non-trivial amount of boilerplate (action creators, selectors, listener middleware for `Appearance.addChangeListener`) for zero benefit. The skill defaults the theme store to Zustand and treats it as an internal implementation detail.

**If `/rn-setup` did NOT pick Zustand** (i.e. Redux or None), install zustand now — only for the theme store:

```bash
yarn add zustand
```

This is the *only* place a Redux-state project mixes in Zustand, and it's intentional: 3 KB, no providers, no boilerplate. Application state stays in Redux; theme state stays here.

```typescript
import { Appearance } from 'react-native';

import { create } from 'zustand';

export type ThemeMode = 'light' | 'dark' | 'system';

export type ThemeStateType = {
  mode: ThemeMode;             // user's preference (incl. 'system')
  theme: 'light' | 'dark';     // resolved theme to apply
  setMode: (mode: ThemeMode) => void;
};

// Appearance.getColorScheme() returns `'light' | 'dark' | null | undefined` in
// current RN typings — never trust it to be one of just the two strings.
const normalize = (scheme: string | null | undefined): 'light' | 'dark' =>
  scheme === 'dark' ? 'dark' : 'light';

const resolve = (mode: ThemeMode): 'light' | 'dark' => {
  if (mode === 'system') {
    return normalize(Appearance.getColorScheme());
  }
  return mode;
};

export const useThemeState = create<ThemeStateType>(set => ({
  mode: 'system',
  theme: resolve('system'),
  setMode: mode => set({ mode, theme: resolve(mode) }),
}));

// Re-resolve when OS theme changes (only if user is in 'system' mode)
Appearance.addChangeListener(({ colorScheme }) => {
  const { mode } = useThemeState.getState();
  if (mode === 'system') {
    useThemeState.setState({ theme: normalize(colorScheme) });
  }
});
```

Update `src/store/index.ts` barrel to export `themeStore` alongside `authStore` and `storage`.

> Default is `'system'` so the app respects OS dark mode automatically. Users can override via `setMode('light')` / `setMode('dark')`.

---

### 4.4 Optional utilities (conditional on Q4)

Generate only the items the user selected in Q4. Skip everything else entirely (no files, no installs).

---

#### 4.4 — Option 1: Deep-linking base

Generic skeleton — handles arrival, parsing, dynamic-pattern matching, and `navigationRef`-based navigation. The user fills in their actual routes; the infrastructure is reusable.

**No new deps** (uses RN's built-in `Linking` API + the existing `@react-navigation/native` install).

Create `src/utils/deepLink/extractPath.ts`:

```typescript
import Config from 'react-native-config';

/**
 * Strip scheme + host from a URL to leave a clean path.
 *
 * Reads `WEB_BASE_URL` from your .env.* files (e.g. https://yourdomain.com)
 * and strips that host plus any custom scheme. Also strips the query string
 * and URL fragment so the result is matchable.
 *
 *   extractPath('https://yourdomain.com/products/aloe?ref=ig#review')
 *     -> '/products/aloe'
 *   extractPath('myapp://products/aloe')
 *     -> '/products/aloe'
 */
export const extractPath = (url: string): string => {
  if (!url) return '/';
  let path = url;

  // Strip web host from WEB_BASE_URL (handles http(s)://… variants)
  const webBase = (Config.WEB_BASE_URL ?? '').replace(/\/$/, '');
  if (webBase) {
    const hostOnly = webBase.replace(/^https?:\/\//, '');
    path = path.replace(new RegExp(`^https?://${hostOnly}`, 'i'), '');
  }

  // Strip any leftover scheme (custom scheme like myapp://, or unmatched http(s)://)
  path = path.replace(/^[a-z][a-z0-9+\-.]*:\/\/[^/]*/i, '');

  // Strip query string and fragment
  path = path.split('?')[0].split('#')[0];

  return path.startsWith('/') ? path : `/${path}`;
};
```

Create `src/utils/deepLink/extractParams.ts`:

```typescript
/** Parse a URL's query string into a plain object. Returns `{}` if no query. */
export const extractParams = (url: string): Record<string, string> => {
  const queryStart = url.indexOf('?');
  if (queryStart === -1) return {};

  const queryString = url.slice(queryStart + 1).split('#')[0];
  const params: Record<string, string> = {};

  for (const pair of queryString.split('&')) {
    if (!pair) continue;
    const [k, v = ''] = pair.split('=');
    params[decodeURIComponent(k)] = decodeURIComponent(v);
  }
  return params;
};
```

Create `src/utils/deepLink/index.ts`:

```typescript
export { extractPath } from './extractPath';
export { extractParams } from './extractParams';
```

Create `src/navigation/navigationRef.ts`:

```typescript
import { createNavigationContainerRef } from '@react-navigation/native';

export const navigationRef = createNavigationContainerRef();
```

Create `src/navigation/routesConfig.ts` — **the user fills this in**:

```typescript
/**
 * Route definition for the deep-link router.
 *
 * Dynamic-only: every path is a pattern. Static paths just have no `:params`.
 *   { pattern: '/men',                screenName: 'Home' }
 *   { pattern: '/products/:handle',   screenName: 'ProductDetail' }
 *   { pattern: '/orders/:id/items',   screenName: 'OrderItems' }
 *
 * Add entries as your app grows. Order matters: the first match wins.
 */
export type RouteDef = {
  pattern: string;
  screenName: string;
};

export const ROUTES_CONFIG: RouteDef[] = [
  // Example — uncomment and adapt:
  // { pattern: '/home',               screenName: 'Home' },
  // { pattern: '/products/:handle',   screenName: 'ProductDetail' },
];
```

Create `src/navigation/resolveRoute.ts`:

```typescript
import { RouteDef } from './routesConfig';

export interface ResolvedRoute {
  screenName: string;
  params: Record<string, string>;
}

/**
 * Match a path against the route table.
 *
 *   resolveRoute('/products/aloe', [{ pattern: '/products/:handle', screenName: 'PDP' }])
 *     -> { screenName: 'PDP', params: { handle: 'aloe' } }
 *
 * Returns null if nothing matches.
 */
export const resolveRoute = (
  path: string,
  routes: RouteDef[],
): ResolvedRoute | null => {
  const pathSegments = path.split('/').filter(Boolean);

  for (const route of routes) {
    const patternSegments = route.pattern.split('/').filter(Boolean);
    if (patternSegments.length !== pathSegments.length) continue;

    const params: Record<string, string> = {};
    let matched = true;

    for (let i = 0; i < patternSegments.length; i++) {
      const p = patternSegments[i];
      if (p.startsWith(':')) {
        params[p.slice(1)] = pathSegments[i];
      } else if (p !== pathSegments[i]) {
        matched = false;
        break;
      }
    }

    if (matched) return { screenName: route.screenName, params };
  }

  return null;
};
```

Create `src/navigation/navigateByPath.ts`:

```typescript
import { Linking } from 'react-native';

import { extractParams, extractPath } from '@/utils/deepLink';

import { navigationRef } from './navigationRef';
import { resolveRoute } from './resolveRoute';
import { ROUTES_CONFIG } from './routesConfig';

/**
 * Route a URL into the navigation stack.
 *
 * - Resolves the URL against ROUTES_CONFIG.
 * - Merges path params (from `:slug` matchers) with query-string params.
 * - Falls back to opening the URL in an external browser if nothing matches.
 *
 * Returns true if a route was matched + navigated, false otherwise.
 *
 * If your project uses nested navigators (e.g. Drawer → Tabs → Screen), edit
 * the navigationRef.navigate(...) call below to your nested shape, e.g.:
 *
 *   navigationRef.navigate('Drawer', {
 *     screen: 'Tabs',
 *     params: { screen: resolved.screenName, params },
 *   });
 */
export const navigateByPath = (url: string): boolean => {
  if (!url) return false;

  const path = extractPath(url);
  const queryParams = extractParams(url);
  const resolved = resolveRoute(path, ROUTES_CONFIG);

  if (!resolved) {
    if (__DEV__) {
      // eslint-disable-next-line no-console
      console.warn(`[deepLink] no route matched for "${path}" (from "${url}")`);
    }
    // Optional: open unknown links in browser. Comment out if you prefer silent no-op.
    Linking.canOpenURL(url).then(can => {
      if (can) Linking.openURL(url);
    });
    return false;
  }

  const params = { ...resolved.params, ...queryParams };
  navigationRef.navigate(resolved.screenName as never, params as never);
  return true;
};
```

Create `src/hooks/useDeepLink.ts`:

```typescript
import { useCallback, useEffect, useRef } from 'react';
import { Linking } from 'react-native';

import { navigateByPath } from '@/navigation/navigateByPath';
import { navigationRef } from '@/navigation/navigationRef';

/**
 * Wires deep-link arrival into the navigation stack.
 *
 * - Cold start: `Linking.getInitialURL()` reads whatever URL launched the app.
 * - Warm start: `Linking.addEventListener('url', …)` catches URLs received
 *   while the app is running.
 * - Race guard: if the navigation stack isn't ready when a URL arrives
 *   (typical for cold start), the URL is queued in `pendingUrlRef` and
 *   flushed by `onNavigationReady` — which the caller wires into
 *   `<NavigationContainer onReady={…} />`.
 *
 * Usage:
 *   const { onNavigationReady } = useDeepLink();
 *   <NavigationContainer ref={navigationRef} onReady={onNavigationReady}>…
 */
export const useDeepLink = () => {
  const pendingUrlRef = useRef<string | null>(null);

  const handleUrl = useCallback((url: string) => {
    if (!navigationRef.isReady()) {
      pendingUrlRef.current = url;
      return;
    }
    navigateByPath(url);
  }, []);

  const onNavigationReady = useCallback(() => {
    const pending = pendingUrlRef.current;
    if (!pending) return;
    pendingUrlRef.current = null;
    navigateByPath(pending);
  }, []);

  useEffect(() => {
    Linking.getInitialURL().then(url => {
      if (url) handleUrl(url);
    });
    const subscription = Linking.addEventListener('url', ({ url }) => handleUrl(url));
    return () => subscription.remove();
  }, [handleUrl]);

  return { onNavigationReady };
};
```

**Update `src/navigation/RootNavigator.tsx`** (or wherever `NavigationContainer` is mounted) — wire up the hook:

```typescript
import { NavigationContainer } from '@react-navigation/native';

import { useDeepLink } from '@/hooks/useDeepLink';
import { navigationRef } from '@/navigation/navigationRef';

export const RootNavigator = () => {
  const { onNavigationReady } = useDeepLink();

  return (
    <NavigationContainer ref={navigationRef} onReady={onNavigationReady}>
      {/* existing stack navigator */}
    </NavigationContainer>
  );
};
```

> **What's intentionally NOT included:**
> - **Native manifest config** (`AndroidManifest.xml` intent filter, `Info.plist` `CFBundleURLTypes`) — these need a chosen URL scheme (e.g. `myapp://`) or a verified Universal Links domain, which most projects don't have at scaffold time. Add them when ready.
> - **Push-payload routing** — push notifications carry deep links too, but FCM setup goes in the future `/rn-push` skill which will hook into this same `navigateByPath`.
> - **Nested-navigator hardcoding** — `navigateByPath.ts` ships with a flat `navigationRef.navigate(screenName, params)` call. Projects with Drawer/Tabs edit the comment-block snippet inside that file (one-line change). Keeping it un-opinionated prevents wrong defaults.

Update barrels: `src/utils/index.ts` (add `./deepLink`), `src/navigation/index.ts` (add `navigationRef`, `navigateByPath`, `RouteDef`, `ROUTES_CONFIG`), `src/hooks/index.ts` (add `useDeepLink`).

---

#### 4.4 — Option 2: Error handler

Create `src/utils/errorHandler.ts`:

```typescript
/**
 * Lightweight error/event reporter with a dev-vs-prod split.
 *
 * In `__DEV__`: logs to the console (preserving stack + context for inspection).
 * In production: silenced by default — wire up Sentry / Crashlytics / your
 * provider of choice in the `reportToProvider` stub below.
 *
 * The shape is designed to swap cleanly for `@sentry/react-native` later
 * (Sentry's `captureException(error, { extra })` accepts the same context
 * object), so the call sites you write today don't change when you adopt
 * `/rn-firebase` or any other observability skill.
 */
export type ErrorContext = Record<string, unknown>;

const reportToProvider = (_error: unknown, _context?: ErrorContext): void => {
  // Replace this body once a provider is configured. Example:
  //   Sentry.captureException(_error, { extra: _context });
  //   crashlytics().recordError(_error as Error);
};

export const errorHandler = (error: unknown, context?: ErrorContext): void => {
  if (__DEV__) {
    // eslint-disable-next-line no-console
    console.error('[errorHandler]', error, context ?? {});
    return;
  }
  reportToProvider(error, context);
};

export const logEvent = (message: string, context?: ErrorContext): void => {
  if (__DEV__) {
    // eslint-disable-next-line no-console
    console.log('[logEvent]', message, context ?? {});
    return;
  }
  // Hook into your analytics here later, e.g. analytics().logEvent(message, context).
};
```

> **Why no Sentry install:** observability is a separate concern. When you adopt `/rn-firebase` (deferred skill), the `reportToProvider` stub becomes a 2-line edit instead of a refactor across the codebase.

Update `src/utils/index.ts` barrel.

---

#### 4.4 — Option 3: Toast manager (store + component)

Two files, always paired.

Install: `react-native-reanimated` is already installed by `/rn-core` Step 1. No new deps.

Create `src/store/toastStore.ts`:

```typescript
import { create } from 'zustand';

export type ToastPosition = 'top' | 'bottom';
export type ToastVariant = 'success' | 'error' | 'info';

interface ToastState {
  message: string | null;
  position: ToastPosition;
  variant: ToastVariant;
  visible: boolean;
  show: (
    message: string,
    options?: { position?: ToastPosition; variant?: ToastVariant; durationMs?: number },
  ) => void;
  hide: () => void;
}

let hideTimer: ReturnType<typeof setTimeout> | null = null;

export const useToastStore = create<ToastState>(set => ({
  message: null,
  position: 'bottom',
  variant: 'info',
  visible: false,

  show: (message, options) => {
    if (hideTimer) {
      clearTimeout(hideTimer);
      hideTimer = null;
    }
    set({
      message,
      position: options?.position ?? 'bottom',
      variant: options?.variant ?? 'info',
      visible: true,
    });
    hideTimer = setTimeout(() => {
      set({ visible: false });
      hideTimer = null;
    }, options?.durationMs ?? 3000);
  },

  hide: () => {
    if (hideTimer) {
      clearTimeout(hideTimer);
      hideTimer = null;
    }
    set({ visible: false });
  },
}));
```

> **Why `zustand` and not Context:** toasts fire from anywhere (utility functions, error handlers, side effects) — `useToastStore.getState().show(…)` works outside React without needing a hook. Context would force every call site to either be a component or pass the dispatcher around.

Create `src/components/core/Toast.tsx`:

```typescript
import { useEffect } from 'react';
import { StyleSheet, Text } from 'react-native';

import Animated, {
  useAnimatedStyle,
  useSharedValue,
  withTiming,
} from 'react-native-reanimated';

import { useThemeStyles } from '@/hooks';
import { spacing, typography } from '@/theme';
import { useToastStore } from '@/store/toastStore';

export const Toast = () => {
  const { message, position, variant, visible } = useToastStore();
  const styles = useToastStyles(variant);
  const opacity = useSharedValue(0);
  const translateY = useSharedValue(20);

  useEffect(() => {
    opacity.value = withTiming(visible ? 1 : 0, { duration: 220 });
    translateY.value = withTiming(visible ? 0 : 20, { duration: 220 });
  }, [visible, opacity, translateY]);

  const animatedStyle = useAnimatedStyle(() => ({
    opacity: opacity.value,
    transform: [{ translateY: position === 'top' ? -translateY.value : translateY.value }],
  }));

  if (!message) return null;

  return (
    <Animated.View
      pointerEvents="none"
      style={[
        styles.container,
        position === 'top' ? styles.top : styles.bottom,
        animatedStyle,
      ]}
    >
      <Text style={styles.message}>{message}</Text>
    </Animated.View>
  );
};

const useToastStyles = (variant: 'success' | 'error' | 'info') =>
  useThemeStyles(
    ({ colors }) => ({
      container: {
        position: 'absolute',
        left: spacing.lg,
        right: spacing.lg,
        paddingVertical: spacing.md,
        paddingHorizontal: spacing.lg,
        borderRadius: 8,
        backgroundColor:
          variant === 'error'
            ? colors.error
            : variant === 'success'
              ? colors.success
              : colors.backgroundInverse,
      },
      top:    { top: spacing.xxl },
      bottom: { bottom: spacing.xxl },
      message: { ...typography.bodySmall, color: colors.textInverse, textAlign: 'center' },
    }),
    [variant],
  );
```

**Mount the Toast in `AppWrapper`** (Step 7). Inside the `BottomSheetModalProvider`, add `<Toast />` as a sibling of `{children}` so it renders above all screens:

```typescript
<BottomSheetModalProvider>
  {children}
  <Toast />
</BottomSheetModalProvider>
```

Update barrels: `src/store/index.ts` (export `useToastStore`), `src/components/core/index.ts` (export `Toast`).

---

#### 4.4 — Option 4: Location permission helper

Install: `react-native-permissions`. Runs `pod install` after.

```bash
yarn add react-native-permissions
cd ios && pod install && cd ..
```

> **iOS Info.plist note:** the user must add `NSLocationWhenInUseUsageDescription` (and `NSLocationAlwaysAndWhenInUseUsageDescription` if they want background) to `ios/<App>/Info.plist` with a human-readable reason string. iOS rejects the build at runtime without it. The skill prints this reminder; it doesn't modify Info.plist.

> **Android Manifest note:** the user must add `<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />` (or `ACCESS_COARSE_LOCATION`) to `android/app/src/main/AndroidManifest.xml`. The skill prints this reminder.

Create `src/hooks/useLocationPermission.ts`:

```typescript
import { useCallback } from 'react';
import { Platform } from 'react-native';

import {
  check,
  PERMISSIONS,
  request,
  RESULTS,
  type PermissionStatus,
} from 'react-native-permissions';

const LOCATION_PERMISSION = Platform.select({
  ios: PERMISSIONS.IOS.LOCATION_WHEN_IN_USE,
  android: PERMISSIONS.ANDROID.ACCESS_FINE_LOCATION,
});

export type LocationPermissionResult = {
  status: PermissionStatus;
  granted: boolean;
};

export const useLocationPermission = () => {
  const checkPermission = useCallback(async (): Promise<LocationPermissionResult> => {
    if (!LOCATION_PERMISSION) {
      return { status: RESULTS.UNAVAILABLE, granted: false };
    }
    const status = await check(LOCATION_PERMISSION);
    return { status, granted: status === RESULTS.GRANTED };
  }, []);

  const requestPermission = useCallback(async (): Promise<LocationPermissionResult> => {
    if (!LOCATION_PERMISSION) {
      return { status: RESULTS.UNAVAILABLE, granted: false };
    }
    const status = await request(LOCATION_PERMISSION);
    return { status, granted: status === RESULTS.GRANTED };
  }, []);

  return { checkPermission, requestPermission };
};
```

> **Why no Android-GPS-enable prompt baked in:** `react-native-permissions` v4 dropped the `promptForEnableLocationIfNeeded` helper. The Knya/Foxtale projects we benchmarked still used it via `react-native-android-location-enabler` (separate dep). Don't install that here — most apps don't need it (the permission flow already takes the user to settings on Android). Add it later if a screen explicitly needs the GPS-on dialog.

Update `src/hooks/index.ts` barrel.

---

#### 4.4 — Option 5: Network status (hook + banner — Variant 2)

Install: `@react-native-community/netinfo`. Run `pod install`.

```bash
yarn add @react-native-community/netinfo
cd ios && pod install && cd ..
```

Create `src/hooks/useNetworkStatus.ts`:

```typescript
import { useEffect, useState } from 'react';
import NetInfo, { NetInfoState } from '@react-native-community/netinfo';

export type NetworkStatus = {
  isConnected: boolean;
  isInternetReachable: boolean | null;
  type: NetInfoState['type'];
};

export const useNetworkStatus = (): NetworkStatus => {
  const [state, setState] = useState<NetworkStatus>({
    isConnected: true,
    isInternetReachable: null,
    type: 'unknown',
  });

  useEffect(() => {
    const unsubscribe = NetInfo.addEventListener(s => {
      setState({
        isConnected: !!s.isConnected,
        isInternetReachable: s.isInternetReachable,
        type: s.type,
      });
    });
    NetInfo.fetch().then(s => {
      setState({
        isConnected: !!s.isConnected,
        isInternetReachable: s.isInternetReachable,
        type: s.type,
      });
    });
    return () => unsubscribe();
  }, []);

  return state;
};
```

> **Why `isConnected` AND `isInternetReachable`:** `isConnected` is true on captive-portal Wi-Fi and other "physically connected but no internet" cases. `isInternetReachable` runs an actual reachability probe. The banner below uses `isConnected === false` (the more conservative signal); a screen that needs to actually hit the network should check `isInternetReachable === false`.

Create `src/components/core/NetworkStatusBanner.tsx`:

```typescript
import { useEffect } from 'react';
import { StyleSheet, Text } from 'react-native';

import Animated, {
  useAnimatedStyle,
  useSharedValue,
  withTiming,
} from 'react-native-reanimated';
import { useSafeAreaInsets } from 'react-native-safe-area-context';

import { useThemeStyles } from '@/hooks';
import { useNetworkStatus } from '@/hooks/useNetworkStatus';
import { spacing, typography } from '@/theme';

/**
 * Slim non-blocking banner at the top of the screen, visible only when offline.
 * Auto-hides on reconnect. Sits above safe-area inset.
 */
export const NetworkStatusBanner = () => {
  const { isConnected } = useNetworkStatus();
  const insets = useSafeAreaInsets();
  const styles = useBannerStyles();
  const translateY = useSharedValue(-100);

  useEffect(() => {
    translateY.value = withTiming(isConnected ? -100 : 0, { duration: 220 });
  }, [isConnected, translateY]);

  const animatedStyle = useAnimatedStyle(() => ({
    transform: [{ translateY: translateY.value }],
  }));

  return (
    <Animated.View
      pointerEvents="none"
      style={[styles.banner, { paddingTop: insets.top + spacing.xs }, animatedStyle]}
    >
      <Text style={styles.text}>No internet — some features may not work</Text>
    </Animated.View>
  );
};

const useBannerStyles = () =>
  useThemeStyles(({ colors }) => ({
    banner: {
      position: 'absolute',
      top: 0,
      left: 0,
      right: 0,
      paddingHorizontal: spacing.lg,
      paddingBottom: spacing.sm,
      backgroundColor: colors.error,
      zIndex: 1000,
    },
    text: {
      ...typography.bodySmall,
      color: colors.textInverse,
      textAlign: 'center',
    },
  }));
```

**Mount the banner in `AppWrapper`** (Step 7). Right after `SafeAreaProvider` opens so the banner can read insets:

```typescript
<SafeAreaProvider>
  <NetworkStatusBanner />
  <BottomSheetModalProvider>{children}</BottomSheetModalProvider>
</SafeAreaProvider>
```

Update barrels: `src/hooks/index.ts` (`useNetworkStatus`), `src/components/core/index.ts` (`NetworkStatusBanner`).

> **Hook is exported for screen-level use too.** High-stakes screens (checkout, payment) can do `const { isInternetReachable } = useNetworkStatus(); if (isInternetReachable === false) return <FullPageOffline />` and render their own blocking UI on top of the global banner.

---

## Step 5 — Core Hooks

### `src/hooks/useThemeStyles.ts`

```typescript
import { useMemo } from 'react';

import { useThemeState, ThemeStateType } from '@/store';
import { ThemeColorsType, ThemeResources } from '@/theme';
import { NamedStyles } from '@/types';

export type UseThemePropsType = ThemeStateType & { colors: ThemeColorsType };

export const useThemeStyles = <T extends NamedStyles<T>>(
  func: (args: UseThemePropsType) => T,
  dependencies: ReadonlyArray<unknown> = [],
): T => {
  const props = useThemeState();
  const colors = ThemeResources[props.theme];
  return useMemo(
    () => func({ ...props, colors }),
    // eslint-disable-next-line react-hooks/exhaustive-deps
    [props, colors, ...dependencies],
  );
};

export const useThemeValues = (): UseThemePropsType => {
  const props = useThemeState();
  const colors = ThemeResources[props.theme];
  return useMemo(() => ({ ...props, colors }), [props, colors]);
};
```

> The `eslint-disable-next-line react-hooks/exhaustive-deps` is intentional: the `dependencies` array is spread in, and exhaustive-deps can't see through the spread. The standard React docs sanction this exact pattern.

No `as any`. Caller's style object infers cleanly:

```typescript
const styles = useThemeStyles(({ colors }) => ({
  container: { backgroundColor: colors.background }, // typed
}));
styles.container; // ViewStyle, fully typed
```

Update `src/hooks/index.ts` barrel.

---

## Step 6 — Core Components

### 6.1 BaseSkeleton — `src/components/skeleton/BaseSkeleton.tsx`

```typescript
import { memo, useEffect, useMemo } from 'react';
import { StyleProp, StyleSheet, ViewStyle } from 'react-native';

import Animated, {
  cancelAnimation,
  interpolateColor,
  useAnimatedStyle,
  useSharedValue,
  withRepeat,
  withTiming,
} from 'react-native-reanimated';

import { useThemeValues } from '@/hooks';

type BaseSkeletonProps = {
  style?: StyleProp<ViewStyle>;
  duration?: number;
  isLoading?: boolean;
};

export const BaseSkeleton = ({
  style,
  duration = 500,
  isLoading = true,
}: BaseSkeletonProps) => {
  const progress = useSharedValue(0);
  const { colors } = useThemeValues();

  const shimmerColors = useMemo(
    () => [colors.skeletonShimmerStart, colors.skeletonShimmerEnd],
    [colors.skeletonShimmerStart, colors.skeletonShimmerEnd],
  );

  useEffect(() => {
    if (isLoading) {
      progress.value = withRepeat(withTiming(1, { duration }), -1, true);
    }
    return () => cancelAnimation(progress);
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [duration, isLoading]);

  const animatedStyle = useAnimatedStyle(
    () => ({
      backgroundColor: interpolateColor(progress.value, [0, 1], shimmerColors),
    }),
    [shimmerColors],
  );

  return isLoading ? (
    <Animated.View style={[skeletonStyles.fill, style, animatedStyle]} />
  ) : null;
};

const skeletonStyles = StyleSheet.create({
  fill: {
    position: 'absolute',
    top: 0,
    left: 0,
    right: 0,
    bottom: 0,
    zIndex: 1000,
  },
});

export const MemoizedBaseSkeleton = memo(BaseSkeleton);
```

> **Why explicit position styles, not `StyleSheet.absoluteFillObject`:** in React Native 0.80+ the StyleSheet typings only expose `StyleSheet.absoluteFill` (a readonly object), not the older `absoluteFillObject`. Spreading `absoluteFill` works at runtime but TypeScript rejects it. Inlining the four `position`/edges in a named StyleSheet entry is the most portable form.
>
> The `eslint-disable-next-line` on the cleanup effect is intentional — the cleanup uses `progress` (a stable Reanimated shared value reference), which exhaustive-deps over-reports.

Add `src/components/skeleton/index.ts` barrel.

### 6.2 FastImage Wrapper — `src/components/core/image/FastImage.tsx`

```typescript
import { useCallback, useMemo, useState } from 'react';
import { StyleProp, StyleSheet, View, ViewStyle } from 'react-native';

import DefaultFastImage, {
  FastImageProps as DefaultFastImageProps,
} from '@d11/react-native-fast-image';

import { MemoizedBaseSkeleton } from '@/components/skeleton';

export type FastImageProps = DefaultFastImageProps & {
  containerStyle?: StyleProp<ViewStyle>;
};

export const FastImage = ({
  containerStyle,
  ...imageProps
}: FastImageProps) => {
  const [isLoading, setIsLoading] = useState(true);
  const [isError, setIsError] = useState(false);

  const showPlaceholder = useMemo(
    () => !imageProps.source || isError,
    [imageProps.source, isError],
  );

  const handleLoadStart = useCallback(() => setIsLoading(true), []);
  const handleLoadEnd = useCallback(() => setIsLoading(false), []);
  const handleError = useCallback(() => {
    setIsError(true);
    setIsLoading(false);
  }, []);

  return (
    <View style={containerStyle}>
      <MemoizedBaseSkeleton isLoading={isLoading} />
      {showPlaceholder ? (
        <View style={styles.placeholder} />
      ) : (
        <DefaultFastImage
          resizeMode={DefaultFastImage.resizeMode.cover}
          onLoadStart={handleLoadStart}
          onLoadEnd={handleLoadEnd}
          onError={handleError}
          style={styles.image}
          {...imageProps}
        />
      )}
    </View>
  );
};

const styles = StyleSheet.create({
  placeholder: { flex: 1, backgroundColor: '#EEEEEE' },
  image: { flex: 1 },
});
```

> Three things changed from the original snippet: (1) handlers are hoisted out of JSX so they get stable identities (calling `useCallback` inline in props means new function refs every render — bypassing memoization on the underlying `<DefaultFastImage>`); (2) inline `{flex: 1}` styles moved into `StyleSheet.create` — required by the `react-native/no-inline-styles` ESLint rule; (3) styles use `StyleSheet.create` rather than the `useThemeStyles` hook because the placeholder colour is intentionally non-themed (it shows briefly while the real image loads and a themed background would still flash).

Add `src/components/core/image/index.ts` and `src/components/core/index.ts` barrels.

### 6.3 BaseList (FlashList Wrapper) — `src/components/list/BaseList.tsx`

```typescript
import { forwardRef, Ref } from 'react';

import { FlashList, FlashListProps, FlashListRef } from '@shopify/flash-list';

const BaseListInner = forwardRef<
  FlashListRef<unknown>,
  FlashListProps<unknown>
>((props, ref) => (
  <FlashList
    bounces={false}
    showsVerticalScrollIndicator={false}
    showsHorizontalScrollIndicator={false}
    ref={ref}
    {...props}
  />
));

export function BaseList<ItemT>(
  props: FlashListProps<ItemT> & { ref?: Ref<FlashListRef<ItemT>> },
) {
  return (
    <BaseListInner
      {...(props as FlashListProps<unknown>)}
      ref={props.ref as Ref<FlashListRef<unknown>>}
    />
  );
}
```

> The cast through `FlashListProps<unknown>` is unavoidable: `forwardRef` can't be generic over `ItemT` directly, so we wrap a non-generic `forwardRef` and re-introduce the generic via a function component. This is the conventional FlashList wrapper pattern.

Add `src/components/list/index.ts` barrel.

**FlashList usage rules** (callers of `BaseList` must follow):
- `estimatedItemSize` is **required** — measure a real item's rendered height and pass it. Wrong values cause scroll jank and white flashes.
- `keyExtractor` must return a stable, unique string for each item. **Never use the array index** — it breaks recycling when the list reorders.
- Wrap the `renderItem` component in `React.memo` so FlashList can skip re-renders for unchanged items.
- Pass `useCallback`-stabilised functions as props to list items, never inline arrow functions.
- Use `FlashList` for every scrollable list. Fall back to `FlatList` or `SectionList` only when FlashList genuinely cannot support the required structure (rare).

### 6.4 ScreenWrapper — `src/components/wrappers/ScreenWrapper.tsx`

```typescript
import { ReactNode, useMemo } from 'react';

import {
  Edge,
  SafeAreaView,
  SafeAreaViewProps,
} from 'react-native-safe-area-context';

import { useThemeStyles } from '@/hooks';

export type ScreenWrapperProps = {
  children?: ReactNode;
  enableTopSafeArea?: boolean;
  enableBottomSafeArea?: boolean;
} & SafeAreaViewProps;

export const ScreenWrapper = ({
  children,
  enableTopSafeArea,
  enableBottomSafeArea,
  ...containerProps
}: ScreenWrapperProps) => {
  const styles = useScreenWrapperStyles();

  const edges = useMemo<Edge[]>(() => {
    const e: Edge[] = ['left', 'right'];
    if (!enableTopSafeArea) {
      e.push('top');
    }
    if (!enableBottomSafeArea) {
      e.push('bottom');
    }
    return e;
  }, [enableTopSafeArea, enableBottomSafeArea]);

  return (
    <>
      {enableTopSafeArea && <SafeAreaView edges={['top']} />}
      <SafeAreaView
        edges={edges}
        {...containerProps}
        style={[styles.container, containerProps?.style]}
      >
        {children}
      </SafeAreaView>
      {enableBottomSafeArea && <SafeAreaView edges={['bottom']} />}
    </>
  );
};

const useScreenWrapperStyles = () =>
  useThemeStyles(({ colors }) => ({
    container: { flex: 1, backgroundColor: colors.background },
  }));
```

> **Why `Edge[]` and not `SafeAreaViewProps['edges']`:** in `react-native-safe-area-context@5+`, `edges` is typed as `readonly Edge[]`, so the original `const e: SafeAreaViewProps['edges'] = [...]; e.push(...)` fails — `.push()` doesn't exist on readonly arrays. Importing the `Edge` type and declaring a plain mutable array is the fix.

### 6.5 KeyboardAwareSVWrapper — `src/components/wrappers/KeyboardAwareSVWrapper.tsx`

```typescript
import { forwardRef } from 'react';
import { StyleSheet } from 'react-native';

import {
  KeyboardAwareScrollView,
  KeyboardAwareScrollViewProps,
  KeyboardAwareScrollViewRef,
} from 'react-native-keyboard-controller';

export const KeyboardAwareSVWrapper = forwardRef<
  KeyboardAwareScrollViewRef,
  KeyboardAwareScrollViewProps
>((props, ref) => (
  <KeyboardAwareScrollView
    keyboardShouldPersistTaps="handled"
    bounces={false}
    bottomOffset={50}
    showsVerticalScrollIndicator={false}
    showsHorizontalScrollIndicator={false}
    {...props}
    ref={ref}
    style={[styles.fill, props?.style]}
  />
));

const styles = StyleSheet.create({
  fill: { flex: 1 },
});
```

> **Important:** for `KeyboardAwareScrollView` to actually respond to the keyboard, the app must mount `<KeyboardProvider>` (from the same package) somewhere above it — see Step 7.

Add `src/components/wrappers/index.ts` barrel.

---

## Step 7 — AppWrapper (Root Providers)

Look up the API-layer choice the user made in `rn-setup` (Q3). Generate the variant that matches:

### Variant A — `rn-setup` chose REST + React Query

```bash
yarn add @tanstack/react-query
```

```typescript
import { PropsWithChildren } from 'react';
import { StyleSheet } from 'react-native';

import { BottomSheetModalProvider } from '@gorhom/bottom-sheet';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { GestureHandlerRootView } from 'react-native-gesture-handler';
import { SafeAreaProvider } from 'react-native-safe-area-context';

const queryClient = new QueryClient({
  defaultOptions: {
    queries: { staleTime: 5 * 60 * 1000, retry: 2 },
  },
});

export const AppWrapper = ({ children }: PropsWithChildren) => (
  <QueryClientProvider client={queryClient}>
    <GestureHandlerRootView style={styles.root}>
      <SafeAreaProvider>
        <BottomSheetModalProvider>{children}</BottomSheetModalProvider>
      </SafeAreaProvider>
    </GestureHandlerRootView>
  </QueryClientProvider>
);

const styles = StyleSheet.create({ root: { flex: 1 } });
```

> **Optional — persist React Query cache across launches (opt-in, NOT scaffolded by default):**
>
> For offline-first browsing or instant cold-start UI, React Query's cache can be persisted to MMKV via `@tanstack/query-async-storage-persister` + `@tanstack/react-query-persist-client`. **The scaffold deliberately does not install this.** Reasons specific to e-commerce:
> - **Stale price / inventory risk.** Persisted product queries can show old prices and "in stock" badges that lie. For a checkout-bound flow this is a real UX (and revenue) problem.
> - **Default `staleTime: 0` works against persistence** — every persisted query refetches on mount anyway, so the cold-start benefit is small unless you also tune `staleTime` and `gcTime` per query.
> - **Encryption overhead.** Persisted cache holds query data that might include auth-flavoured fields; encrypting it adds CPU on every dehydrate/hydrate cycle.
>
> If you genuinely want it, the production-shape pattern is **selective persistence** — only allowlist queries that are safe to serve stale (user profile, app settings, last-viewed-collection IDs), not everything. Sketch:
>
> ```typescript
> import { createAsyncStoragePersister } from '@tanstack/query-async-storage-persister';
> import { persistQueryClient } from '@tanstack/react-query-persist-client';
>
> import { getStorage } from '@/store/storage';
>
> const mmkvAsAsyncStorage = {
>   getItem: (key: string) => Promise.resolve(getStorage().getString(key) ?? null),
>   setItem: (key: string, value: string) => Promise.resolve(getStorage().set(key, value)),
>   removeItem: (key: string) => Promise.resolve(getStorage().remove(key)),
> };
>
> persistQueryClient({
>   queryClient,
>   persister: createAsyncStoragePersister({ storage: mmkvAsAsyncStorage }),
>   dehydrateOptions: {
>     shouldDehydrateQuery: q => {
>       const allowlist = ['user-profile', 'app-settings'];
>       return allowlist.includes(q.queryKey[0] as string);
>     },
>   },
> });
> ```
>
> Add this AFTER `bootstrapStorage()` resolves (otherwise `getStorage()` throws). And install `@tanstack/query-async-storage-persister @tanstack/react-query-persist-client` first.

### Variant B — `rn-setup` chose GraphQL (Apollo) or `None`

Do NOT install `@tanstack/react-query` (it's dead weight when Apollo owns the cache, and inflates the bundle). Strip `QueryClientProvider` from `AppWrapper`:

```typescript
import { PropsWithChildren } from 'react';
import { StyleSheet } from 'react-native';

import { BottomSheetModalProvider } from '@gorhom/bottom-sheet';
import { GestureHandlerRootView } from 'react-native-gesture-handler';
import { SafeAreaProvider } from 'react-native-safe-area-context';

export const AppWrapper = ({ children }: PropsWithChildren) => (
  <GestureHandlerRootView style={styles.root}>
    <SafeAreaProvider>
      <BottomSheetModalProvider>{children}</BottomSheetModalProvider>
    </SafeAreaProvider>
  </GestureHandlerRootView>
);

const styles = StyleSheet.create({ root: { flex: 1 } });
```

> **Why two variants:** the previous template installed `@tanstack/react-query` unconditionally. If `rn-setup` chose Apollo, that package is unused and the `QueryClientProvider` is a no-op — better to remove both than ship dead code. If the user genuinely needs both libraries side-by-side later (Apollo for GraphQL, React Query for REST), they can `yarn add @tanstack/react-query` and re-add the provider.

### Update `App.tsx`

Wrap the root navigator with `AppWrapper`, **and** mount `<KeyboardProvider>` (from `react-native-keyboard-controller`) so `KeyboardAwareSVWrapper` works. The provider order is:

1. API provider (`ApolloProvider` or skip for REST — React Query is inside `AppWrapper`)
2. `KeyboardProvider`
3. `AppWrapper` (GestureHandler → SafeArea → BottomSheetModal)
4. `NavigationContainer`
5. `RootNavigator`

Example (Apollo / GraphQL flow):

```typescript
import { ApolloProvider } from '@apollo/client/react';
import { NavigationContainer } from '@react-navigation/native';
import { KeyboardProvider } from 'react-native-keyboard-controller';

import { AppWrapper } from '@/components';
import { RootNavigator } from '@/navigation';
import { apolloClient } from '@/services';

function App() {
  return (
    <ApolloProvider client={apolloClient}>
      <KeyboardProvider>
        <AppWrapper>
          <NavigationContainer>
            <RootNavigator />
          </NavigationContainer>
        </AppWrapper>
      </KeyboardProvider>
    </ApolloProvider>
  );
}

export default App;
```

> Apollo is outermost so any descendant can call Apollo hooks. `KeyboardProvider` must mount somewhere above any `KeyboardAwareScrollView`/`KeyboardAvoidingView` from the library — putting it just inside the API provider keeps reach maximal without polluting the API context.

### Storage bootstrap gate (Q3 = Yes only)

Encrypted storage from sub-step 4.1 reads the key from the Keychain asynchronously, so the app must wait for it before rendering anything that reads from storage. Add a gate to `App.tsx`:

**Zustand variant:**

```typescript
import { useEffect, useState } from 'react';

import { ApolloProvider } from '@apollo/client/react';
import { NavigationContainer } from '@react-navigation/native';
import { KeyboardProvider } from 'react-native-keyboard-controller';

import { AppWrapper } from '@/components';
import { RootNavigator } from '@/navigation';
import { apolloClient } from '@/services';
import { bootstrapStorage } from '@/store/storage';
import { useAuthStore } from '@/store';

function App() {
  const [storageReady, setStorageReady] = useState(false);

  useEffect(() => {
    (async () => {
      await bootstrapStorage();
      await useAuthStore.persist.rehydrate();
      setStorageReady(true);
    })();
  }, []);

  if (!storageReady) {
    return null; // Or a splash component — render time is ~50–150ms in practice
  }

  return (
    <ApolloProvider client={apolloClient}>
      <KeyboardProvider>
        <AppWrapper>
          <NavigationContainer>
            <RootNavigator />
          </NavigationContainer>
        </AppWrapper>
      </KeyboardProvider>
    </ApolloProvider>
  );
}

export default App;
```

**Redux variant:**

```typescript
import { useEffect, useState } from 'react';

import { ApolloProvider } from '@apollo/client/react';
import { NavigationContainer } from '@react-navigation/native';
import { KeyboardProvider } from 'react-native-keyboard-controller';
import { Provider as ReduxProvider } from 'react-redux';
import { PersistGate } from 'redux-persist/integration/react';

import { AppWrapper } from '@/components';
import { RootNavigator } from '@/navigation';
import { apolloClient } from '@/services';
import { persistor, store } from '@/store/store';
import { bootstrapStorage } from '@/store/storage';

function App() {
  const [storageReady, setStorageReady] = useState(false);

  useEffect(() => {
    bootstrapStorage().then(() => setStorageReady(true));
  }, []);

  if (!storageReady) {
    return null;
  }

  return (
    <ApolloProvider client={apolloClient}>
      <ReduxProvider store={store}>
        <PersistGate loading={null} persistor={persistor}>
          <KeyboardProvider>
            <AppWrapper>
              <NavigationContainer>
                <RootNavigator />
              </NavigationContainer>
            </AppWrapper>
          </KeyboardProvider>
        </PersistGate>
      </ReduxProvider>
    </ApolloProvider>
  );
}

export default App;
```

> **Two gates, not one.** `bootstrapStorage()` resolves once the encryption key is loaded — that's when `redux-persist` can safely read MMKV. The `<PersistGate>` then waits for `redux-persist` to finish rehydrating the persisted slices before children render. Skipping either gate causes a flash of empty state and, worse, racy writes against an unbootstrapped store.

**Q3 = No:** skip the bootstrap gate entirely. Storage is ready synchronously; render the providers directly as in the example at the top of this step.

---

## Step 8 — Example Home Screen

Replace (or create) `src/screens/Home/HomeScreen.tsx` with this themed example so future screens have a reference pattern:

```typescript
import { Text, View } from 'react-native';

import { ScreenWrapper } from '@/components';
import { useThemeStyles } from '@/hooks';
import { spacing, typography } from '@/theme';

export const HomeScreen = () => {
  const styles = useStyles();
  return (
    <ScreenWrapper>
      <View style={styles.body}>
        <Text style={styles.title}>Welcome</Text>
        <Text style={styles.subtitle}>
          This screen is themed via useThemeStyles. Edit src/screens/Home/HomeScreen.tsx to get started.
        </Text>
      </View>
    </ScreenWrapper>
  );
};

const useStyles = () =>
  useThemeStyles(({ colors }) => ({
    body: { padding: spacing.lg, gap: spacing.sm },
    title: { ...typography.h2, color: colors.text },
    subtitle: { ...typography.body, color: colors.textSecondary },
  }));
```

This demonstrates the recommended pattern: `ScreenWrapper` for safe-area + bg, `useThemeStyles` for themed styles, `spacing` + `typography` from `@/theme`.

Update `src/screens/Home/index.ts` and `src/screens/index.ts` barrels.

---

## Step 9 — Update Barrel Exports

Ensure every folder under `src/` has an `index.ts` that re-exports its contents:

- `src/components/index.ts` — exports from `core/`, `skeleton/`, `list/`, `wrappers/`, `wrapper/`
- `src/hooks/index.ts` — exports `useThemeStyles`, `useThemeValues`
- `src/theme/index.ts` — exports `ThemeResources`, `ThemeColorsType`, `typography`, `fontFamily`, `spacing`, `SpacingKey`, `FlexBoxStyles`
- `src/types/index.ts` — exports `NamedStyles`
- `src/store/index.ts` — extends the rn-setup barrel to re-export `themeStore` alongside `authStore` and `storage`

---

## Step 10 — iOS Pod Install + Verify

```bash
cd ios && pod install && cd ..
yarn lint --fix     # autofix import order, surface any rule violations
npx tsc --noEmit    # MUST exit 0 — catches Edge[], ThemeColorsType, ColorScheme drift
yarn test           # MUST exit 0 — passWithNoTests:true covers the empty case
yarn ios            # or yarn android
```

All three checks (`lint`, `tsc`, `test`) must pass before declaring done. If `tsc` reports errors, **stop** — they almost always indicate a version-API drift in one of the snippets (StyleSheet, Edge, ColorScheme, Apollo subpath, etc.). Fix in this skill, then re-run.

If a font was configured in Step 0, also verify text renders in that font (drop the `.ttf` files first if you haven't).

---

## Rules

- Use the project's package manager for ALL installs
- Do NOT change bundle ID or applicationId
- All imports MUST use `@/` aliases
- Every folder MUST have barrel `index.ts`
- **Themed styles MUST use `useThemeStyles`.** Static styles that never depend on theme (e.g. `flex: 1`, absolute-fill shells, placeholder colours that show only briefly) are fine in `StyleSheet.create`. Never write inline `style={{ ... }}` object literals in JSX — that triggers the `react-native/no-inline-styles` rule and breaks referential equality on every render.
- No anonymous functions in JSX props (use `useCallback` + named handlers) — critical for FlashList item performance. Don't call `useCallback` *inline* in a JSX prop either; hoist the handler above the `return`.
- If the user replied `skip` to the font question, do NOT add any font package or `.ttf` reference
- If the project already has any of these files, skip — do not overwrite
- Run `yarn lint --fix`, `npx tsc --noEmit`, and `yarn test` at the end — all three must pass

---
name: rn-setup
description: Interactive scaffold for a production-ready React Native CLI project. Asks preferences (package manager, state, API, etc.) then sets up everything.
allowed-tools: Bash Read Write Edit Glob Grep
---

# React Native Project Setup — Interactive Scaffold

This skill sets up a production-ready React Native CLI project by walking the user through key decisions, then generating everything in one shot.

**IMPORTANT:** Before doing anything, ask the user the questions below. Do NOT assume answers. Wait for all responses, then execute the full setup.

---

## Step 1 — Ask Preferences

Ask the user these questions in a single message. Present them as a numbered list so the user can reply quickly (e.g. "1, 2, 1, 1, 1"):

### Q1. Package Manager
1. **Yarn Classic** (recommended — best RN ecosystem support)
2. **Bun** (fastest, but may hit edge cases with RN native builds)
3. **npm** (default, slower)

### Q2. State Management
1. **Zustand** — lightweight, minimal boilerplate, great for small-medium apps (auth store persisted via MMKV)
2. **Redux Toolkit** — scalable, excellent devtools, built-in async (createAsyncThunk)
3. **None** — skip for now

### Q3. API Layer
1. **REST (Axios + React Query)** — for REST backends, with caching and retry
2. **GraphQL (Apollo Client)** — for GraphQL backends, with Apollo cache
3. **None** — skip for now

### Q4. Navigation
1. **React Navigation v7** (recommended — industry standard)
2. **None** — skip for now

### Q5. Environment Config
1. **Yes** — set up `.env.dev/.stage/.prod` with `react-native-config`
2. **No** — skip for now

---

## Step 2 — Execute Setup

After receiving answers, execute ALL applicable steps below. Skip sections the user opted out of. Only create what's missing — if something already exists, skip it.

### 2.1 Package Manager Switch

If not already using the chosen package manager:

**Yarn:**
```bash
rm -rf node_modules package-lock.json
yarn install
```
Add `"packageManager": "yarn@1.22.22"` to package.json.

**Bun:**
```bash
rm -rf node_modules package-lock.json
bun install
```

**npm:** No changes needed (default).

After switching, use the chosen package manager for ALL subsequent installs.

### 2.2 Folder Structure

Create under `src/`:
```
src/
  components/
  screens/
  navigation/
  hooks/
  services/
  store/
  utils/
  constants/
  theme/
  types/
  assets/
    images/
    fonts/
    svgs/
```

Add `index.ts` with `export {};` in each folder as barrel file placeholder.

### 2.3 Path Aliases + TypeScript Strict

Install `babel-plugin-module-resolver` (dev dep).

Update `babel.config.js`:
```js
plugins: [
  ['module-resolver', { root: ['./src'], alias: { '@': './src' } }]
]
```

Update `tsconfig.json`:
```json
"compilerOptions": {
  "baseUrl": ".",
  "paths": { "@/*": ["src/*"] },
  "strict": true
}
```

`strict: true` enables strictNullChecks, noImplicitAny, and friends — required to enforce the "no `any`" rule.

### 2.4 Code Quality (ESLint + Prettier + Husky + Testing)

Install dev deps: `eslint-plugin-import`, `eslint-import-resolver-typescript`, `husky`, `lint-staged@^16`, `@testing-library/react-native`

> **Why `lint-staged@^16`:** v17 declares `"engines": { "node": ">=22.22.1" }`. Many machines run Node 22.x at minor versions below `.22.1` (which is the `.nvmrc` we pin in 2.5 as `22`). Yarn errors out on that engine mismatch. v16 has the same feature surface and works on any Node 22.

**ESLint** — update `.eslintrc.js`:
- Add `eslint-plugin-import` with `plugin:import/recommended` + `plugin:import/typescript` extends, `import/resolver.typescript` set to `./tsconfig.json`
- `import/order` rule: external (react/react-native first) > builtin > internal (@/**) > parent/sibling > type. Alphabetize, newlines between groups.
- `react-hooks/exhaustive-deps`: `warn` (NOT off — turning it off hides stale closure bugs)
- `@typescript-eslint/no-unused-vars`: `error` with `ignoreRestSiblings: true`, `argsIgnorePattern: '^_'`, `varsIgnorePattern: '^_'`

**Prettier** — ensure `.prettierrc.js` has: `singleQuote: true`, `trailingComma: 'all'`, `bracketSpacing: true`

**Husky + lint-staged:**
- Add `"prepare": "husky"` to scripts
- Add lint-staged config to package.json (eslint --fix + prettier --write on ts/tsx/js/jsx; prettier --write on json/md)
- Run `npx husky init`, set `.husky/pre-commit` to `npx lint-staged`

**Testing** — `@testing-library/react-native` is installed. Update `jest.config.js`:
```js
module.exports = {
  preset: '@react-native/jest-preset',
  moduleNameMapper: {
    '^@/(.*)$': '<rootDir>/src/$1',
  },
  passWithNoTests: true,
};
```

> **Why `moduleNameMapper`:** the RN preset uses `babel-jest`, so the `@/` Babel alias from 2.3 *would* transform import paths — but Jest's module resolver runs before Babel for matching, so unmatched `@/foo` resolves to nothing and tests fail with `Cannot find module '@/foo'`. The explicit mapper points Jest at the same `src/` root.
>
> **Why `passWithNoTests: true`:** we delete the template `__tests__/App.test.tsx` below, and `jest` exits 1 on an empty test run otherwise. The flag flips that to 0. Real tests (added per the rules below) override the no-tests case anyway.

Add a coverage script in `package.json`:
```json
"test:coverage": "jest --coverage"
```

**Delete the template App snapshot test:**

```bash
rm __tests__/App.test.tsx
```

> The default RN template ships an `App.test.tsx` that renders the entire `<App />` tree with `react-test-renderer`. That worked for a bare template, but once `App.tsx` is wrapped with `ApolloProvider` / `NavigationContainer` / `GestureHandlerRootView` / MMKV-backed Zustand stores (all native modules), the snapshot test cascades into needing mocks for 6+ libraries. The maintenance burden isn't worth it for a placeholder test. The convention below (one test per hook/util/screen) is what catches regressions.

**Testing rules** (enforce in code review):
- Every custom hook in `src/hooks/` must have a unit test.
- Every utility function in `src/utils/` must have a unit test.
- Screen components: snapshot test as the minimum; add interaction tests for forms.
- Mock at the boundary (Apollo, native modules, network) — never mock internal modules.
- Tests live in `__tests__/` mirroring the `src/` structure.

### 2.5 Dev Environment Consistency

Create `.nvmrc` at the repo root with:
```
22
```

This pins the Node version for everyone using `nvm use` / `fnm use`.

Create `.editorconfig` at the repo root with:
```
root = true

[*]
indent_style = space
indent_size = 2
end_of_line = lf
charset = utf-8
trim_trailing_whitespace = true
insert_final_newline = true

[*.md]
trim_trailing_whitespace = false
```

This enforces consistent indentation, line endings, and charset across every teammate's IDE.

### 2.6 Environment (if chosen)

Install `react-native-config`.

Create `.env.dev`, `.env.stage`, `.env.prod` with:
```
API_BASE_URL=
APP_DISPLAY_NAME=
```

Create `.env.example` as committed template.

Create `src/constants/env.ts`:
```typescript
import Config from 'react-native-config';

export const ENV = {
  API_BASE_URL: Config.API_BASE_URL ?? '',
  APP_DISPLAY_NAME: Config.APP_DISPLAY_NAME ?? '',
};
```

Add `.env.*` and `!.env.example` to `.gitignore`.

### 2.7 Navigation (if chosen)

Install: `@react-navigation/native`, `@react-navigation/native-stack`, `react-native-screens`

(`react-native-safe-area-context` should already be present; install if missing)

Create `src/navigation/types.ts`:
```typescript
export type RootStackParamList = {
  Home: undefined;
};

declare global {
  namespace ReactNavigation {
    interface RootParamList extends RootStackParamList {}
  }
}
```

Create `src/screens/Home/HomeScreen.tsx` — minimal screen with View + Text.

Create `src/navigation/RootNavigator.tsx` — native stack with HomeScreen.

Update barrel exports in `src/navigation/index.ts` and `src/screens/index.ts`.

### 2.8 State Management (if chosen)

**Zustand (with MMKV persistence):**

Install: `zustand`, `react-native-mmkv`, `react-native-nitro-modules`

> `react-native-nitro-modules` is a required peer dep of `react-native-mmkv`. Always install them together.

Create `src/store/storage.ts`:
```typescript
import { createMMKV } from 'react-native-mmkv';
import type { StateStorage } from 'zustand/middleware';

export const mmkv = createMMKV({
  id: 'app-storage',
  encryptionKey: 'app-storage-key', // override per-project; for sensitive prod use, fetch from Keychain/Keystore at boot
});

export const zustandMMKVStorage: StateStorage = {
  getItem: name => mmkv.getString(name) ?? null,
  setItem: (name, value) => mmkv.set(name, value),
  removeItem: name => {
    mmkv.remove(name);
  },
};
```

> **MMKV v4 API:** the constructor moved from `new MMKV({...})` to `createMMKV({...})` (it now returns a Nitro hybrid object, not a class instance). Likewise the delete method is `remove(key)` rather than `delete(key)`. If you're pinned to MMKV v2/v3 instead, use the old API.

Create `src/store/authStore.ts`:
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
    (set) => ({
      token: null,
      user: null,
      isAuthenticated: false,
      setAuth: (token, user) => set({ token, user, isAuthenticated: true }),
      clearAuth: () => set({ token: null, user: null, isAuthenticated: false }),
    }),
    {
      name: 'auth-storage',
      storage: createJSONStorage(() => zustandMMKVStorage),
    },
  ),
);
```

Auth survives app restarts via MMKV. The same `mmkv` instance can be reused for any other persisted store.

**Redux Toolkit:**

Install: `@reduxjs/toolkit`, `react-redux`

Create `src/store/store.ts` with `configureStore`.
Create `src/store/hooks.ts` with typed `useAppDispatch` and `useAppSelector`.
Create `src/store/slices/authSlice.ts` with token, user, login/logout thunks.

> Redux persistence (`redux-persist` with an MMKV adapter) is out of scope for this skill — wire it manually if needed.

Update barrel in `src/store/index.ts`.

### 2.9 API Layer (if chosen)

**REST (Axios + React Query):**
Install: `axios`, `@tanstack/react-query`

Create `src/services/axiosClient.ts`:
- baseURL from `ENV.API_BASE_URL`
- Request interceptor: inject auth token from store
- Response interceptor: 401 → clear auth

Create `src/services/queryClient.ts`:
- 5 min stale time, 2 retries, 10 min gcTime

**GraphQL (Apollo):**
Install: `@apollo/client`, `graphql`, `rxjs`

> **Why `rxjs`:** Apollo Client v4 (current major) replaced its `zen-observable` internals with RxJS and declares `rxjs@^7.3.0` as a peer dependency. Yarn Classic does NOT auto-install peers — without it, queries/mutations throw at runtime the first time they construct an observable. (If you pin `@apollo/client@^3`, you can drop `rxjs`.)

Create `src/services/apolloClient.ts`:
```typescript
import { ApolloClient, HttpLink, InMemoryCache } from '@apollo/client';
import { setContext } from '@apollo/client/link/context';

import { ENV } from '@/constants';
import { useAuthStore } from '@/store';

const httpLink = new HttpLink({ uri: ENV.API_BASE_URL });

const authLink = setContext((_, { headers }) => {
  const token = useAuthStore.getState().token;
  return {
    headers: {
      ...headers,
      ...(token ? { authorization: `Bearer ${token}` } : {}),
    },
  };
});

export const apolloClient = new ApolloClient({
  link: authLink.concat(httpLink),
  cache: new InMemoryCache(),
});
```

> **Apollo v4 import paths:** `ApolloProvider` ships from `@apollo/client/react` (not `@apollo/client`). React bindings are split into their own subpath so the core client can be tree-shaken in non-React environments. Make sure App.tsx in 2.10 uses `import { ApolloProvider } from '@apollo/client/react'`.

Update barrel in `src/services/index.ts`.

### 2.10 Wire Up App.tsx

Update `App.tsx` to wrap with the chosen providers in this order (outer → inner):
1. `QueryClientProvider` (from `@tanstack/react-query`) or `ApolloProvider` (from `@apollo/client/react`) (if API chosen)
2. Redux `Provider` (if Redux chosen)
3. `SafeAreaProvider`
4. `NavigationContainer` (if navigation chosen)
5. `RootNavigator` or existing app content

> Apollo `ApolloProvider` import: `from '@apollo/client/react'` (NOT `@apollo/client`) — see 2.9.

**IMPORTANT: Remove template packages that are no longer used after rewiring:**

```bash
yarn remove @react-native/new-app-screen
```

This package is part of the default RN template and is always replaced by the custom scaffold. Always remove it.

### 2.11 iOS Pods

Run `pod install` in the `ios/` directory to link new native modules.

### 2.12 Final Verification & Cleanup

Run all three checks before declaring the scaffold done — silence on any of these is the only acceptable success signal:

```bash
yarn lint --fix                # autofix import order, etc.
npx tsc --noEmit               # MUST exit 0 — catches API drift like MMKV v4
yarn test                      # MUST exit 0 — with passWithNoTests: true this passes the empty case
```

- Remove any unused packages from dependencies (the only one this scaffold typically removes is `@react-native/new-app-screen`, already covered in 2.10)
- `yarn lint` may report `import/no-named-as-default` warnings for `react-native-config` and (if you kept it) `react-test-renderer` — both are default-only exports. These warnings are benign; leave them.

If `tsc` reports errors, do NOT proceed — they almost always indicate a version-API mismatch (e.g. MMKV constructor, Apollo subpath, ColorScheme typing). Fix the snippet locally before continuing.

### 2.13 Update CLAUDE.md

Update CLAUDE.md to reflect:
- Commands using the chosen package manager
- Folder structure
- Component tree with actual provider wrappers
- Navigation, state, API layer docs
- Conventions
- Key dependencies list

---

## Rules

- Always use the package manager the user chose for ALL installs
- Do NOT change bundle ID or applicationId
- Use TypeScript everywhere — no `any` types (enforced via `tsconfig.json` `strict: true`)
- All imports must use `@/` path aliases
- Every `src/` folder must have a barrel `index.ts`
- Screens = UI only; business logic in hooks/services
- No anonymous functions in JSX props (e.g. `onPress={handlePress}`, not `onPress={() => doSomething()}`) — they break referential equality and cause unnecessary re-renders, especially in lists
- Run lint fix at the end to enforce import ordering
- Remove unused packages and dead code
- Keep CLAUDE.md in sync with all changes

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

Install dev deps: `eslint-plugin-import`, `eslint-import-resolver-typescript`, `husky`, `lint-staged`, `@testing-library/react-native`

**ESLint** — update `.eslintrc.js`:
- Add `eslint-plugin-import`
- `import/order` rule: external (react/react-native first) > builtin > internal (@/**) > parent/sibling > type. Alphabetize, newlines between groups.
- `react-hooks/exhaustive-deps`: `warn` (NOT off — turning it off hides stale closure bugs)
- `@typescript-eslint/no-unused-vars`: `error` with `ignoreRestSiblings: true`

**Prettier** — ensure `.prettierrc.js` has: `singleQuote: true`, `trailingComma: 'all'`, `bracketSpacing: true`

**Husky + lint-staged:**
- Add `"prepare": "husky"` to scripts
- Add lint-staged config to package.json
- Run `npx husky init`, set `.husky/pre-commit` to `npx lint-staged`

**Testing** — `@testing-library/react-native` is installed. The RN template's Jest preset is enabled by default in package.json (`"jest": { "preset": "@react-native/jest-preset" }`). Add a coverage script:
```json
"test:coverage": "jest --coverage"
```

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
import { MMKV } from 'react-native-mmkv';
import type { StateStorage } from 'zustand/middleware';

export const mmkv = new MMKV({ id: 'app-storage' });

export const zustandMMKVStorage: StateStorage = {
  getItem: (name) => mmkv.getString(name) ?? null,
  setItem: (name, value) => mmkv.set(name, value),
  removeItem: (name) => mmkv.delete(name),
};
```

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
Install: `@apollo/client`, `graphql`

Create `src/services/apolloClient.ts`:
- HttpLink with `ENV.API_BASE_URL`
- authLink to inject token
- InMemoryCache

Update barrel in `src/services/index.ts`.

### 2.10 Wire Up App.tsx

Update `App.tsx` to wrap with the chosen providers in this order (outer → inner):
1. `QueryClientProvider` or `ApolloProvider` (if API chosen)
2. Redux `Provider` (if Redux chosen)
3. `SafeAreaProvider`
4. `NavigationContainer` (if navigation chosen)
5. `RootNavigator` or existing app content

**IMPORTANT: Remove template packages that are no longer used after rewiring:**

```bash
yarn remove @react-native/new-app-screen
```

This package is part of the default RN template and is always replaced by the custom scaffold. Always remove it.

### 2.11 iOS Pods

Run `pod install` in the `ios/` directory to link new native modules.

### 2.12 Final Cleanup

- Run `yarn lint --fix` (or equivalent) to auto-fix import ordering
- Remove any unused packages from dependencies
- Verify lint passes clean

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
- Run lint fix at the end to enforce import ordering
- Remove unused packages and dead code
- Keep CLAUDE.md in sync with all changes

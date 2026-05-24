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

Ask the user these two questions before any installs. They control the theme and typography setup. Present them in a single message:

### Q1. Brand Colors
"Do you have brand colors for this project? Provide hex codes (e.g. `primary: #FF6B00, secondary: #1B1B1B`) or reply `skip` to use a neutral black/white default palette."

- Parse any provided colors into named slots (`primary`, `secondary`, `accent`, etc.).
- If `skip` or empty, use the default palette below.

### Q2. Font Family
"Which font family does this project use? Provide a name (e.g. `Inter`, `Roboto`) or reply `skip` to use the system default font."

- If a name is provided, set the `fontFamily` constants to `<Name>-Regular`, `<Name>-Medium`, `<Name>-SemiBold`, `<Name>-Bold`. After setup, tell the user to drop the corresponding `.ttf` files into `src/assets/fonts/` and run `npx react-native-asset` (or set up `react-native.config.js`) to link them.
- If `skip`, set every `fontFamily` value to `undefined` so React Native uses the system font. Do NOT install any font package.

Wait for both answers before proceeding.

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

> `react-native-nitro-modules` is a required peer dependency of `react-native-mmkv`. Always install it alongside mmkv. MMKV supports its own encryption via the `encryptionKey` option — sufficient for most apps. Add `react-native-encrypted-storage` separately only if you need OS-level keychain/keystore for biometric tokens or similar.

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

## Step 4 — Zustand Theme Store (with OS dark mode integration)

### `src/store/themeStore.ts`

> **Why `src/store/` and not `src/zustand/`:** `rn-setup` already created `src/store/` for Zustand stores (`authStore.ts`, `storage.ts`). Adding a separate `src/zustand/` folder would split state across two parallel locations. Naming this file `themeStore.ts` mirrors `authStore.ts`; the exported hook is `useThemeState` to match `useAuthStore`.

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

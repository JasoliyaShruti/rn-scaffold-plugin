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

### Animation & Gesture

```bash
yarn add react-native-reanimated react-native-gesture-handler
```

> `react-native-worklets` is bundled inside reanimated v4+ as a transitive dep — no separate install needed.

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
} as const;

export const LightThemeColors = {
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

export type ThemeColorsType = typeof LightThemeColors;
```

Create `src/theme/resources/DarkThemeResources.ts` — same shape, concrete dark values:

```typescript
const STATIC_COLORS_DARK = {
  black: '#111111',
  white: '#FFFFFF',
  transparent: 'transparent',
  // Same brand colors as light if provided
} as const;

export const DarkThemeColors = {
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
import { LightThemeColors } from './LightThemeResources';
import { DarkThemeColors } from './DarkThemeResources';

export const ThemeResources = {
  light: LightThemeColors,
  dark: DarkThemeColors,
} as const;

export type { ThemeColorsType } from './LightThemeResources';
```

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

### `src/zustand/ThemeState.ts`

```typescript
import { Appearance } from 'react-native';
import { create } from 'zustand';

export type ThemeMode = 'light' | 'dark' | 'system';

export type ThemeStateType = {
  mode: ThemeMode;             // user's preference (incl. 'system')
  theme: 'light' | 'dark';     // resolved theme to apply
  setMode: (mode: ThemeMode) => void;
};

const resolve = (mode: ThemeMode): 'light' | 'dark' => {
  if (mode === 'system') return Appearance.getColorScheme() ?? 'light';
  return mode;
};

export const ThemeState = create<ThemeStateType>((set) => ({
  mode: 'system',
  theme: resolve('system'),
  setMode: (mode) => set({ mode, theme: resolve(mode) }),
}));

// Re-resolve when OS theme changes (only if user is in 'system' mode)
Appearance.addChangeListener(({ colorScheme }) => {
  const { mode } = ThemeState.getState();
  if (mode === 'system') {
    ThemeState.setState({ theme: colorScheme ?? 'light' });
  }
});
```

Update `src/zustand/index.ts` barrel.

> Default is `'system'` so the app respects OS dark mode automatically. Users can override via `setMode('light')` / `setMode('dark')`.

---

## Step 5 — Core Hooks

### `src/hooks/useThemeStyles.ts`

```typescript
import { useMemo } from 'react';

import { ThemeColorsType, ThemeResources } from '@/theme';
import { NamedStyles } from '@/types';
import { ThemeState, ThemeStateType } from '@/zustand';

export type UseThemePropsType = ThemeStateType & { colors: ThemeColorsType };

export const useThemeStyles = <T extends NamedStyles<T>>(
  func: (args: UseThemePropsType) => T,
  dependencies: ReadonlyArray<unknown> = [],
): T => {
  const props = ThemeState();
  const colors = ThemeResources[props.theme];
  return useMemo(
    () => func({ ...props, colors }),
    [props, colors, ...dependencies],
  );
};

export const useThemeValues = (): UseThemePropsType => {
  const props = ThemeState();
  const colors = ThemeResources[props.theme];
  return useMemo(() => ({ ...props, colors }), [props, colors]);
};
```

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

export const BaseSkeleton = ({ style, duration = 500, isLoading = true }: BaseSkeletonProps) => {
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
  }, [duration, isLoading]);

  const animatedStyle = useAnimatedStyle(
    () => ({ backgroundColor: interpolateColor(progress.value, [0, 1], shimmerColors) }),
    [shimmerColors],
  );

  return isLoading ? (
    <Animated.View style={[{ ...StyleSheet.absoluteFillObject, zIndex: 1000 }, style, animatedStyle]} />
  ) : null;
};

export const MemoizedBaseSkeleton = memo(BaseSkeleton);
```

Add `src/components/skeleton/index.ts` barrel.

### 6.2 FastImage Wrapper — `src/components/core/image/FastImage.tsx`

```typescript
import { useCallback, useMemo, useState } from 'react';
import { StyleProp, View, ViewStyle } from 'react-native';
import DefaultFastImage, { FastImageProps as DefaultFastImageProps } from '@d11/react-native-fast-image';
import { MemoizedBaseSkeleton } from '@/components/skeleton';

export type FastImageProps = DefaultFastImageProps & {
  containerStyle?: StyleProp<ViewStyle>;
};

export const FastImage = ({ containerStyle, ...imageProps }: FastImageProps) => {
  const [isLoading, setIsLoading] = useState(true);
  const [isError, setIsError] = useState(false);

  const showPlaceholder = useMemo(() => !imageProps.source || isError, [imageProps.source, isError]);

  return (
    <View style={containerStyle}>
      <MemoizedBaseSkeleton isLoading={isLoading} />
      {showPlaceholder ? (
        <View style={{ flex: 1, backgroundColor: '#EEEEEE' }} />
      ) : (
        <DefaultFastImage
          resizeMode={DefaultFastImage.resizeMode.cover}
          onLoadStart={useCallback(() => setIsLoading(true), [])}
          onLoadEnd={useCallback(() => setIsLoading(false), [])}
          onError={useCallback(() => { setIsError(true); setIsLoading(false); }, [])}
          style={{ flex: 1 }}
          {...imageProps}
        />
      )}
    </View>
  );
};
```

Add `src/components/core/image/index.ts` and `src/components/core/index.ts` barrels.

### 6.3 BaseList (FlashList Wrapper) — `src/components/list/BaseList.tsx`

```typescript
import { forwardRef } from 'react';
import { FlashList, FlashListProps, FlashListRef } from '@shopify/flash-list';

const BaseListInner = forwardRef<FlashListRef<unknown>, FlashListProps<unknown>>((props, ref) => (
  <FlashList
    bounces={false}
    showsVerticalScrollIndicator={false}
    showsHorizontalScrollIndicator={false}
    ref={ref}
    {...props}
  />
));

export function BaseList<ItemT>(
  props: FlashListProps<ItemT> & { ref?: React.Ref<FlashListRef<ItemT>> },
) {
  return (
    <BaseListInner
      {...(props as FlashListProps<unknown>)}
      ref={props.ref as React.Ref<FlashListRef<unknown>>}
    />
  );
}
```

Add `src/components/list/index.ts` barrel.

### 6.4 ScreenWrapper — `src/components/wrappers/ScreenWrapper.tsx`

```typescript
import { useMemo } from 'react';
import { SafeAreaView, SafeAreaViewProps } from 'react-native-safe-area-context';
import { useThemeStyles } from '@/hooks';

export type ScreenWrapperProps = {
  children?: React.ReactNode;
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

  const edges = useMemo(() => {
    const e: SafeAreaViewProps['edges'] = ['left', 'right'];
    if (!enableTopSafeArea) e.push('top');
    if (!enableBottomSafeArea) e.push('bottom');
    return e;
  }, [enableTopSafeArea, enableBottomSafeArea]);

  return (
    <>
      {enableTopSafeArea && <SafeAreaView edges={['top']} />}
      <SafeAreaView edges={edges} {...containerProps} style={[styles.container, containerProps?.style]}>
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

### 6.5 KeyboardAwareSVWrapper — `src/components/wrappers/KeyboardAwareSVWrapper.tsx`

```typescript
import { forwardRef } from 'react';
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
    style={[{ flex: 1 }, props?.style]}
  />
));
```

Add `src/components/wrappers/index.ts` barrel.

---

## Step 7 — AppWrapper (Root Providers)

### `src/components/wrapper/AppWrapper.tsx`

```typescript
import { PropsWithChildren } from 'react';
import { BottomSheetModalProvider } from '@gorhom/bottom-sheet';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { GestureHandlerRootView } from 'react-native-gesture-handler';
import { SafeAreaProvider } from 'react-native-safe-area-context';

const gestureHandlerStyle = { flex: 1 };

const queryClient = new QueryClient({
  defaultOptions: {
    queries: { staleTime: 5 * 60 * 1000, retry: 2 },
  },
});

export const AppWrapper = ({ children }: PropsWithChildren) => (
  <QueryClientProvider client={queryClient}>
    <GestureHandlerRootView style={gestureHandlerStyle}>
      <SafeAreaProvider>
        <BottomSheetModalProvider>
          {children}
        </BottomSheetModalProvider>
      </SafeAreaProvider>
    </GestureHandlerRootView>
  </QueryClientProvider>
);
```

Update `App.tsx` to wrap the root navigator with `AppWrapper`.

---

## Step 8 — Example Home Screen

Replace (or create) `src/screens/Home/HomeScreen.tsx` with this themed example so future screens have a reference pattern:

```typescript
import { Text, View } from 'react-native';
import { ScreenWrapper } from '@/components/wrappers';
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
- `src/theme/index.ts` — exports everything
- `src/types/index.ts` — exports `NamedStyles`
- `src/zustand/index.ts` — exports `ThemeState`

---

## Step 10 — iOS Pod Install + Verify Build

```bash
cd ios && pod install && cd ..
yarn ios  # or yarn android
```

Verify the app builds and runs without errors. If a font was configured in Step 0, also verify text renders in that font (drop the `.ttf` files first if you haven't).

---

## Rules

- Use the project's package manager for ALL installs
- Do NOT change bundle ID or applicationId
- All imports MUST use `@/` aliases
- Every folder MUST have barrel `index.ts`
- Styles via `useThemeStyles` hook — never `StyleSheet.create` in components
- If the user replied `skip` to the font question, do NOT add any font package or `.ttf` reference
- If the project already has any of these files, skip — do not overwrite
- Run `yarn lint --fix` after all files are created

# Phase 1: Foundation - Research

**Researched:** 2026-03-28
**Domain:** Expo SDK 54 project scaffolding, file-based navigation, Catppuccin theming, custom fonts, shared component library, Maestro test infrastructure
**Confidence:** HIGH

## Summary

Phase 1 establishes the mobile app's skeleton: a working Expo development build on iOS and Android, file-based navigation with expo-router, synchronous Catppuccin theming (13 themes matching the web GUI), custom font loading (Plus Jakarta Sans, JetBrains Mono), a shared component library of UI primitives, and Maestro E2E test infrastructure.

The critical path is: project scaffolding with EAS Build pipeline first (FOUND-01, FOUND-02), then theme system with MMKV for synchronous loading (FOUND-04, FOUND-05), then font loading gated behind splash screen (FOUND-06), then navigation structure (FOUND-03), then component library (FOUND-07), and finally Maestro test setup (FOUND-08). The theme system must be implemented before components because every component reads colors from the theme.

**Primary recommendation:** Use `create-expo-app --template tabs` for SDK 54 scaffolding, MMKV + Zustand persist for synchronous theme loading, and extract all 13 theme color tokens directly from the existing web CSS custom properties to guarantee visual parity.

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|-----------------|
| FOUND-01 | App builds and runs on iOS via EAS Build development build | EAS Build setup with development profile, eas.json config, iOS provisioning |
| FOUND-02 | App builds and runs on Android via EAS Build development build | EAS Build setup with development profile, android signing |
| FOUND-03 | File-based navigation (expo-router) with typed routes for all screens | expo-router v4 file conventions, typed routes via experiments.typedRoutes, _layout.tsx pattern |
| FOUND-04 | Catppuccin Mocha theme loads synchronously before first frame (MMKV) | react-native-mmkv synchronous reads, zustand-mmkv-storage adapter, SplashScreen gate |
| FOUND-05 | All 13 themes render identically to web CSS custom properties | CSS token extraction from styles.css, MyrlinTheme interface, automated parity verification |
| FOUND-06 | Custom fonts load before first render | expo-font + SplashScreen.preventAutoHideAsync pattern, @expo-google-fonts/plus-jakarta-sans, bundled JetBrains Mono |
| FOUND-07 | Shared component library matches web design language | 19 core UI primitives (Button, Badge, Card, etc.), themed via useTheme hook, Storybook for visual dev |
| FOUND-08 | Maestro test infrastructure runs on iOS Simulator with screenshot capture | Maestro CLI + idb-companion, YAML flow tests, takeScreenshot command, dev build on simulator |
</phase_requirements>

## Standard Stack

### Core (Phase 1 specific)

| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| Expo SDK 54 | ~54.x | Cross-platform framework | Latest stable. New Architecture default. Last SDK with Legacy Architecture fallback. |
| React Native | 0.81 | UI runtime | Bundled with SDK 54 |
| TypeScript | 5.x | Type safety | Expo default, non-negotiable for this project size |
| expo-router | ~4.x | File-based routing | Bundled with SDK 54, typed routes, deep linking |
| react-native-mmkv | 3.x | Synchronous theme storage | 30x faster than AsyncStorage, sync reads prevent theme flash |
| zustand | 5.0.x | Theme store, UI state | Hooks-based, 1KB, persist middleware with MMKV adapter |
| expo-font | ~13.x | Custom font loading | Bundled, loads Plus Jakarta Sans and JetBrains Mono |
| @expo-google-fonts/plus-jakarta-sans | 0.x | Sans-serif font files | All weights (400, 500, 600, 700) |
| expo-splash-screen | ~0.29.x | Prevent FOUT | Gates first render until fonts + theme ready |
| react-native-reanimated | 4.3.x | Animations (skeleton loaders) | Required for New Architecture, UI thread animations |
| react-native-gesture-handler | 2.x | Touch gestures | Foundation for all interactive components |
| react-native-worklets | (peer dep) | Worklet runtime | Required peer dep for Reanimated 4 |
| expo-dev-client | ~4.x | Development builds | Replaces Expo Go (required for MMKV native module) |

### Dev Tooling

| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| Maestro | latest CLI | E2E visual testing | YAML-based flows, screenshot capture |
| @storybook/react-native | 8.x | Component development | Visual component gallery during dev |
| Jest | 29.x | Unit tests | Bundled with Expo |
| @testing-library/react-native | 12.x | Component tests | Test rendering and interactions |
| @biomejs/biome | 2.x | Lint + format | 10-100x faster than ESLint + Prettier |

### Alternatives Considered

| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| MMKV | AsyncStorage | 30x slower, async-only causes theme flash on app launch |
| Biome | ESLint + Prettier | Biome is faster; either works, soft preference |
| Maestro | Detox | Detox has steeper learning curve, slower; Maestro YAML is agent-writable |
| Zustand | Redux Toolkit | More boilerplate, heavier; Zustand covers our needs |

**Installation (Phase 1 only):**

```bash
# Initialize Expo project with tabs template (SDK 54)
npx create-expo-app@latest myrlin-mobile --template tabs
cd myrlin-mobile

# Core state + storage
npx expo install zustand react-native-mmkv

# Animation + gestures (needed for component library)
npx expo install react-native-reanimated react-native-worklets react-native-gesture-handler

# Fonts
npx expo install expo-font @expo-google-fonts/plus-jakarta-sans

# Dev dependencies
npm install -D @storybook/react-native @testing-library/react-native @biomejs/biome typescript

# Create development build (required for MMKV native module)
npx expo prebuild
eas build --profile development --platform ios
eas build --profile development --platform android
```

## Architecture Patterns

### Recommended Project Structure (Phase 1)

```
mobile/
  app/                          # expo-router screens (file-based routing)
    (tabs)/                     # Bottom tab navigator
      sessions/
        index.tsx               # Placeholder session list
      tasks/
        index.tsx               # Placeholder task board
      costs/
        index.tsx               # Placeholder cost dashboard
      docs/
        index.tsx               # Placeholder docs list
      more/
        index.tsx               # Placeholder more menu
      _layout.tsx               # Tab navigator layout
    _layout.tsx                 # Root layout (ThemeProvider, fonts, splash)
  components/
    ui/                         # Design system primitives
      Button.tsx
      Badge.tsx
      StatusDot.tsx
      Card.tsx
      Input.tsx
      Toggle.tsx
      SegmentedControl.tsx
      ActionSheet.tsx
      EmptyState.tsx
      SectionHeader.tsx
      Skeleton.tsx
      Toast.tsx
      SearchBar.tsx
      MetaRow.tsx
      TokenBar.tsx
      ModalSheet.tsx
      Chip.tsx
      DraggableList.tsx
      index.ts                  # Barrel export
  hooks/
    useTheme.ts                 # Theme context hook
  stores/
    theme-store.ts              # Zustand + MMKV persist
  theme/
    tokens.ts                   # All 13 Catppuccin theme objects
    fonts.ts                    # Font family + size constants
    types.ts                    # MyrlinTheme interface
  types/
    navigation.ts               # Typed route params
  assets/
    fonts/
      JetBrainsMono-Regular.ttf
      JetBrainsMono-Medium.ttf
      JetBrainsMono-Bold.ttf
  maestro/
    flows/
      app-launch.yaml           # Basic launch + screenshot test
    screenshots/                # Captured screenshots
  .storybook/
    main.ts                     # Storybook config
```

### Pattern 1: Synchronous Theme Loading with MMKV + Zustand

**What:** Theme state stored in MMKV (sync) and exposed via Zustand. Theme is available before first render.
**When:** App startup. This is THE critical pattern for FOUND-04.

```typescript
// stores/theme-store.ts
import { MMKV } from 'react-native-mmkv';
import { create } from 'zustand';
import { persist, createJSONStorage } from 'zustand/middleware';
import { themes } from '../theme/tokens';
import type { MyrlinTheme } from '../theme/types';

const storage = new MMKV();

// Zustand MMKV storage adapter (synchronous under the hood)
const mmkvStorage = {
  getItem: (name: string) => {
    const value = storage.getString(name);
    return value ?? null;
  },
  setItem: (name: string, value: string) => {
    storage.set(name, value);
  },
  removeItem: (name: string) => {
    storage.delete(name);
  },
};

interface ThemeState {
  themeId: string;
  theme: MyrlinTheme;
  setTheme: (id: string) => void;
}

export const useThemeStore = create<ThemeState>()(
  persist(
    (set) => ({
      themeId: 'mocha',
      theme: themes.mocha,
      setTheme: (id: string) => {
        const newTheme = themes[id] ?? themes.mocha;
        set({ themeId: id, theme: newTheme });
      },
    }),
    {
      name: 'theme-storage',
      storage: createJSONStorage(() => mmkvStorage),
      // Only persist the themeId, reconstruct theme object on hydration
      partialize: (state) => ({ themeId: state.themeId }),
      onRehydrateStorage: () => (state) => {
        if (state) {
          state.theme = themes[state.themeId] ?? themes.mocha;
        }
      },
    }
  )
);
```

### Pattern 2: Font + Splash Screen Gate

**What:** SplashScreen stays visible until fonts are loaded. Prevents FOUT.
**When:** App startup. Critical for FOUND-06.

```typescript
// app/_layout.tsx
import { SplashScreen, Stack } from 'expo-router';
import { useFonts } from 'expo-font';
import {
  PlusJakartaSans_400Regular,
  PlusJakartaSans_500Medium,
  PlusJakartaSans_600SemiBold,
  PlusJakartaSans_700Bold,
} from '@expo-google-fonts/plus-jakarta-sans';
import { useEffect } from 'react';
import { useThemeStore } from '../stores/theme-store';

// Prevent splash from hiding before fonts load
SplashScreen.preventAutoHideAsync();

export default function RootLayout() {
  const theme = useThemeStore((s) => s.theme);

  const [fontsLoaded, fontError] = useFonts({
    'PlusJakartaSans-Regular': PlusJakartaSans_400Regular,
    'PlusJakartaSans-Medium': PlusJakartaSans_500Medium,
    'PlusJakartaSans-SemiBold': PlusJakartaSans_600SemiBold,
    'PlusJakartaSans-Bold': PlusJakartaSans_700Bold,
    'JetBrainsMono-Regular': require('../assets/fonts/JetBrainsMono-Regular.ttf'),
    'JetBrainsMono-Medium': require('../assets/fonts/JetBrainsMono-Medium.ttf'),
    'JetBrainsMono-Bold': require('../assets/fonts/JetBrainsMono-Bold.ttf'),
  });

  useEffect(() => {
    if (fontsLoaded || fontError) {
      SplashScreen.hideAsync();
    }
  }, [fontsLoaded, fontError]);

  if (!fontsLoaded && !fontError) {
    return null;
  }

  return (
    <ThemeProvider theme={theme}>
      <Stack screenOptions={{ headerShown: false }}>
        <Stack.Screen name="(tabs)" />
      </Stack>
    </ThemeProvider>
  );
}
```

### Pattern 3: Theme Context Provider

**What:** React Context that distributes theme tokens to all components via a `useTheme()` hook.
**When:** Every component that needs colors, spacing, typography, or animation values.

```typescript
// hooks/useTheme.ts
import { createContext, useContext } from 'react';
import type { MyrlinTheme } from '../theme/types';

const ThemeContext = createContext<MyrlinTheme | null>(null);

export function ThemeProvider({ theme, children }: { theme: MyrlinTheme; children: React.ReactNode }) {
  return <ThemeContext.Provider value={theme}>{children}</ThemeContext.Provider>;
}

export function useTheme(): MyrlinTheme {
  const theme = useContext(ThemeContext);
  if (!theme) throw new Error('useTheme must be used within ThemeProvider');
  return theme;
}
```

### Pattern 4: Typed Routes with expo-router

**What:** Enable TypeScript route typing for compile-time navigation safety.
**When:** All navigation. Critical for FOUND-03.

```json
// app.json (partial)
{
  "expo": {
    "experiments": {
      "typedRoutes": true
    }
  }
}
```

After running `npx expo start`, Expo CLI generates a `.expo/types/router.d.ts` file that provides autocomplete for all `<Link href="...">` and `router.push()` calls.

### Pattern 5: Component Library with Theme Integration

**What:** Every shared component reads colors from `useTheme()`, never hardcodes hex values.
**When:** All UI components. Critical for FOUND-07.

```typescript
// components/ui/Button.tsx
import { Pressable, Text, StyleSheet, ActivityIndicator } from 'react-native';
import { useTheme } from '../../hooks/useTheme';
import type { MyrlinTheme } from '../../theme/types';

interface ButtonProps {
  variant?: 'primary' | 'ghost' | 'danger';
  size?: 'sm' | 'md';
  loading?: boolean;
  disabled?: boolean;
  icon?: React.ReactNode;
  children: React.ReactNode;
  onPress?: () => void;
}

export function Button({
  variant = 'primary',
  size = 'md',
  loading = false,
  disabled = false,
  icon,
  children,
  onPress,
}: ButtonProps) {
  const theme = useTheme();
  const styles = getStyles(theme, variant, size);

  return (
    <Pressable
      style={({ pressed }) => [
        styles.container,
        pressed && styles.pressed,
        disabled && styles.disabled,
      ]}
      disabled={disabled || loading}
      onPress={onPress}
    >
      {loading ? (
        <ActivityIndicator size="small" color={styles.text.color} />
      ) : (
        <>
          {icon}
          <Text style={styles.text}>{children}</Text>
        </>
      )}
    </Pressable>
  );
}

function getStyles(theme: MyrlinTheme, variant: string, size: string) {
  const bgMap = {
    primary: theme.colors.accent,
    ghost: 'transparent',
    danger: theme.colors.red,
  };
  const textMap = {
    primary: theme.colors.crust,
    ghost: theme.colors.text,
    danger: theme.colors.crust,
  };
  const paddingMap = {
    sm: { paddingVertical: 6, paddingHorizontal: 12 },
    md: { paddingVertical: 10, paddingHorizontal: 16 },
  };

  return StyleSheet.create({
    container: {
      backgroundColor: bgMap[variant as keyof typeof bgMap],
      borderRadius: theme.radius.md,
      flexDirection: 'row',
      alignItems: 'center',
      justifyContent: 'center',
      gap: theme.spacing.sm,
      ...paddingMap[size as keyof typeof paddingMap],
    },
    pressed: { opacity: 0.85 },
    disabled: { opacity: 0.5 },
    text: {
      color: textMap[variant as keyof typeof textMap],
      fontFamily: theme.typography.fontSans + '-SemiBold',
      fontSize: size === 'sm' ? theme.typography.sizes.sm : theme.typography.sizes.md,
    },
  });
}
```

### Anti-Patterns to Avoid

- **Hardcoded hex colors:** Every color must come from `useTheme()`. Hardcoded values break theme switching and make parity testing impossible.
- **AsyncStorage for theme:** Causes 100-300ms white flash on every app launch. Use MMKV exclusively.
- **Inline StyleSheet.create in render:** Creates new objects every render. Create styles outside the render function or memoize.
- **Skipping SplashScreen gate:** Without `preventAutoHideAsync()`, fonts load after first frame causing visible text reflow (FOUT).

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Theme persistence | Custom file-based storage | MMKV + Zustand persist adapter | MMKV handles encryption, migration, corruption recovery |
| Font loading orchestration | Manual Promise.all with fetch | expo-font useFonts hook | Handles caching, error recovery, platform differences |
| Tab navigation | Custom tab bar from scratch | expo-router (tabs) layout | Handles back button, deep links, gestures, animations |
| Splash screen timing | setTimeout-based delay | expo-splash-screen preventAutoHideAsync | Properly coordinates with native splash, avoids race conditions |
| E2E testing framework | Custom WebDriver scripts | Maestro YAML flows | Zero instrumentation, screenshot capture, CI integration |
| Animation runtime | JS-based requestAnimationFrame | Reanimated 4 worklets | Runs on UI thread, 60fps even during JS execution |
| MMKV-Zustand bridge | Custom adapter from scratch | zustand-mmkv-storage package OR simple adapter (shown above) | Handles serialization, type safety, hydration detection |

**Key insight:** Phase 1 is almost entirely "glue" work connecting well-established libraries. The risk is not in any single library but in the integration order and the development build pipeline setup.

## Common Pitfalls

### Pitfall 1: Development Build Pipeline Takes Longer Than Expected

**What goes wrong:** MMKV requires native code, which means Expo Go is unusable. First EAS Build takes 15-30 minutes and may fail on iOS provisioning.
**Why it happens:** iOS code signing requires Apple Developer account, provisioning profiles, and certificates. Android needs signing keys.
**How to avoid:** Set up EAS Build as the very first task. Create both iOS and Android development builds before writing any app code. Budget 1-2 hours for initial pipeline setup.
**Warning signs:** `npx expo prebuild` fails, `eas build` errors on provisioning, MMKV crashes on import in Expo Go.

### Pitfall 2: SplashScreen.preventAutoHideAsync() Race Condition

**What goes wrong:** Splash screen auto-hides despite calling `preventAutoHideAsync()`, causing a flash of unstyled content.
**Why it happens:** `preventAutoHideAsync()` must be called at module scope (top of file), not inside a component body or useEffect. If called too late, the native splash has already dismissed.
**How to avoid:** Call `SplashScreen.preventAutoHideAsync()` as a top-level statement in `app/_layout.tsx`, outside any component. Call `hideAsync()` only after both fonts AND theme are ready.
**Warning signs:** Brief white flash on app launch, especially on cold start.

### Pitfall 3: MMKV Crashes in Expo Go

**What goes wrong:** App crashes immediately with "Cannot find module react-native-mmkv" or a native module error.
**Why it happens:** MMKV has native (C++) code that cannot run in Expo Go. It requires `npx expo prebuild` and a development build.
**How to avoid:** Never attempt to run with Expo Go after adding MMKV. Always use `eas build --profile development` or local `npx expo run:ios`.
**Warning signs:** Red error screen mentioning missing native module on app launch.

### Pitfall 4: Theme Token Mismatch Between Web and Mobile

**What goes wrong:** Colors look different on mobile vs web, especially in non-Mocha themes.
**Why it happens:** Manually transcribing hex values from CSS to TypeScript introduces typos. Some themes have semantic overrides (border-subtle, shadows) that differ per theme.
**How to avoid:** Write a script that parses `styles.css` and generates the `tokens.ts` file automatically. Cross-verify at least 3 themes (Mocha, Latte, Nord) pixel-for-pixel using screenshots.
**Warning signs:** Subtle color differences only visible in side-by-side comparison. Border opacity mismatches.

### Pitfall 5: JetBrains Mono Not in expo-google-fonts

**What goes wrong:** Developer tries to install `@expo-google-fonts/jetbrains-mono` and it fails or loads wrong font.
**Why it happens:** Although JetBrains Mono is on Google Fonts, the expo-google-fonts package for it exists but may have version issues. More reliable to bundle the .ttf files directly.
**How to avoid:** Download JetBrains Mono .ttf files from the official JetBrains website. Place in `assets/fonts/`. Load via `expo-font` with `require()`. Note: the expo-google-fonts repo does include JetBrains Mono, but bundling gives more control over which weights are included.
**Warning signs:** Font name mismatch in StyleSheet (the fontFamily string must match exactly what was registered).

### Pitfall 6: Maestro iOS Requires Mac with Xcode + idb-companion

**What goes wrong:** Developer tries to run Maestro on Windows or without idb-companion installed.
**Why it happens:** Maestro iOS testing requires: (a) macOS with Xcode, (b) iOS Simulator, (c) Facebook IDB (`brew install facebook/fb/idb-companion`). Cannot run on Windows.
**How to avoid:** Run Maestro tests on the Mac Mini (arthurs-mac-mini / 100.111.181.106). Install idb-companion there. Build the iOS simulator build via `eas build --profile development-simulator --platform ios`. Install on simulator via `xcrun simctl install booted <app.app>`.
**Warning signs:** Maestro hangs or errors about device not found. Note: macOS Sequoia + Xcode 16.2 has known Maestro compatibility issues; check Maestro version and idb-companion version.

## Code Examples

### Theme Token Object (Mocha, extracted from web CSS)

```typescript
// theme/tokens.ts
import type { MyrlinTheme } from './types';

export const mocha: MyrlinTheme = {
  id: 'mocha',
  name: 'Catppuccin Mocha',
  isDark: true,
  colors: {
    base: '#1e1e2e',
    mantle: '#181825',
    crust: '#11111b',
    surface0: '#313244',
    surface1: '#45475a',
    surface2: '#585b70',
    overlay0: '#6c7086',
    overlay1: '#7f849c',
    text: '#cdd6f4',
    subtext0: '#a6adc8',
    subtext1: '#bac2de',
    mauve: '#cba6f7',
    blue: '#89b4fa',
    green: '#a6e3a1',
    yellow: '#f9e2af',
    red: '#f38ba8',
    peach: '#fab387',
    teal: '#94e2d5',
    sky: '#89dceb',
    pink: '#f5c2e7',
    lavender: '#b4befe',
    flamingo: '#f2cdcd',
    rosewater: '#f5e0dc',
    sapphire: '#74c7ec',
    // Semantic aliases
    accent: '#cba6f7',       // = mauve
    success: '#a6e3a1',      // = green
    warning: '#f9e2af',      // = yellow
    error: '#f38ba8',        // = red
    info: '#89b4fa',         // = blue
    // UI semantic
    bgPrimary: '#1e1e2e',    // = base
    bgSecondary: '#181825',  // = mantle
    bgTertiary: '#11111b',   // = crust
    bgElevated: '#313244',   // = surface0
    borderSubtle: 'rgba(69, 71, 90, 0.5)', // = surface1 @ 50%
    borderDefault: '#45475a', // = surface1
    textPrimary: '#cdd6f4',  // = text
    textSecondary: '#bac2de', // = subtext1
    textTertiary: '#a6adc8', // = subtext0
    textMuted: '#7f849c',    // = overlay1
  },
  spacing: { xs: 4, sm: 8, md: 16, lg: 24, xl: 32, xxl: 48 },
  radius: { sm: 6, md: 10, lg: 14, xl: 18, full: 9999 },
  typography: {
    fontSans: 'PlusJakartaSans',
    fontMono: 'JetBrainsMono',
    sizes: { xs: 10, sm: 12, md: 14, lg: 16, xl: 20, xxl: 24 },
    weights: { regular: '400', medium: '500', semibold: '600', bold: '700' },
  },
  shadows: {
    sm: { shadowColor: '#000', shadowOffset: { width: 0, height: 1 }, shadowOpacity: 0.2, shadowRadius: 2, elevation: 2 },
    md: { shadowColor: '#000', shadowOffset: { width: 0, height: 4 }, shadowOpacity: 0.25, shadowRadius: 12, elevation: 4 },
    lg: { shadowColor: '#000', shadowOffset: { width: 0, height: 8 }, shadowOpacity: 0.35, shadowRadius: 30, elevation: 8 },
  },
  animation: { fast: 150, normal: 200, slow: 300, easing: [0.16, 1, 0.3, 1] },
};

// All 13 themes: mocha (default), macchiato, frappe, latte,
// nord, dracula, tokyoNight, cherry, ocean, amber, mint,
// rosePineDawn, gruvboxLight
export const themes: Record<string, MyrlinTheme> = {
  mocha,
  macchiato: { /* ... same structure, values from :root[data-theme="macchiato"] */ },
  frappe: { /* ... */ },
  latte: { /* ... isDark: false */ },
  nord: { /* ... */ },
  dracula: { /* ... */ },
  tokyoNight: { /* ... */ },
  cherry: { /* ... */ },
  ocean: { /* ... */ },
  amber: { /* ... */ },
  mint: { /* ... */ },
  rosePineDawn: { /* ... isDark: false */ },
  gruvboxLight: { /* ... isDark: false */ },
};
```

### 13 Web Themes (source of truth)

The web CSS defines these 13 themes. The mobile tokens.ts MUST match these exactly:

| Theme ID (CSS) | Theme ID (mobile) | isDark | Source |
|---|---|---|---|
| (default :root) | mocha | true | Lines 20-80 of styles.css |
| latte | latte | false | Line 7457 |
| frappe | frappe | true | Line 7585 |
| macchiato | macchiato | true | Line 7650 |
| cherry | cherry | true | ~Line 7720 |
| ocean | ocean | true | ~Line 7790 |
| amber | amber | true | ~Line 7860 |
| mint | mint | true | ~Line 7930 |
| nord | nord | true | ~Line 8000 |
| dracula | dracula | true | ~Line 8070 |
| tokyo-night | tokyoNight | true | ~Line 8140 |
| rose-pine-dawn | rosePineDawn | false | ~Line 8210 |
| gruvbox-light | gruvboxLight | false | ~Line 8280 |

**Recommendation:** Write a Node.js script (`scripts/extract-themes.js`) that parses `src/web/public/styles.css` and auto-generates `mobile/theme/tokens.ts`. This eliminates transcription errors and makes theme updates propagate automatically.

### EAS Build Configuration

```json
// eas.json
{
  "build": {
    "development": {
      "developmentClient": true,
      "distribution": "internal",
      "ios": {
        "simulator": false
      },
      "android": {
        "buildType": "apk"
      }
    },
    "development-simulator": {
      "developmentClient": true,
      "distribution": "internal",
      "ios": {
        "simulator": true
      }
    },
    "preview": {
      "distribution": "internal"
    },
    "production": {}
  }
}
```

### Maestro Basic Flow Test

```yaml
# maestro/flows/app-launch.yaml
appId: com.myrlin.mobile
---
- launchApp:
    clearState: true
- assertVisible: "Sessions"
- takeScreenshot: app-launch-sessions-tab
- tapOn: "Tasks"
- assertVisible: "Tasks"
- takeScreenshot: app-launch-tasks-tab
- tapOn: "Costs"
- takeScreenshot: app-launch-costs-tab
- tapOn: "Docs"
- takeScreenshot: app-launch-docs-tab
- tapOn: "More"
- takeScreenshot: app-launch-more-tab
```

### expo-router Tab Layout

```typescript
// app/(tabs)/_layout.tsx
import { Tabs } from 'expo-router';
import { useTheme } from '../../hooks/useTheme';

export default function TabLayout() {
  const theme = useTheme();

  return (
    <Tabs
      screenOptions={{
        tabBarActiveTintColor: theme.colors.accent,
        tabBarInactiveTintColor: theme.colors.overlay1,
        tabBarStyle: {
          backgroundColor: theme.colors.mantle,
          borderTopColor: theme.colors.surface0,
        },
        headerStyle: {
          backgroundColor: theme.colors.base,
        },
        headerTintColor: theme.colors.text,
        headerTitleStyle: {
          fontFamily: 'PlusJakartaSans-SemiBold',
        },
      }}
    >
      <Tabs.Screen name="sessions" options={{ title: 'Sessions' }} />
      <Tabs.Screen name="tasks" options={{ title: 'Tasks' }} />
      <Tabs.Screen name="costs" options={{ title: 'Costs' }} />
      <Tabs.Screen name="docs" options={{ title: 'Docs' }} />
      <Tabs.Screen name="more" options={{ title: 'More' }} />
    </Tabs>
  );
}
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Expo Go for development | expo-dev-client (dev builds) | SDK 53+ (2025) | Required for native modules like MMKV, push notifications |
| AsyncStorage for preferences | MMKV for sync storage | 2024-2025 | Eliminates theme flash, 30x faster |
| Reanimated 3 | Reanimated 4 | March 2026 | Requires New Architecture; ground-up rewrite for TurboModules |
| FlatList | FlashList v2 | 2025 | JS-only, no size estimates needed, 60fps on complex items |
| expo-barcode-scanner | expo-camera (built-in QR) | SDK 51 (2024) | Deprecated package removed |
| ESLint + Prettier | Biome | 2025-2026 | Single tool, 10-100x faster |

**Deprecated/outdated:**
- `expo-barcode-scanner`: Removed in SDK 51. Use `expo-camera` with `barcodeScannerSettings`.
- `@react-native-async-storage/async-storage`: Still works but should NOT be used for theme/preferences due to async-only API.
- Reanimated 3: Works on SDK 54 but misses New Architecture optimizations.

## Open Questions

1. **Maestro + macOS Sequoia compatibility**
   - What we know: There are reported issues with Maestro + macOS Sequoia 15.0 + Xcode 16.2. The idb-companion (v1.1.8) predates both.
   - What's unclear: Whether Arthur's Mac Mini runs Sequoia, and whether the latest Maestro version has fixes.
   - Recommendation: Test Maestro on the Mac Mini early. If it fails, use EAS Workflows for CI-based Maestro runs instead of local execution.

2. **JetBrains Mono bundling vs expo-google-fonts**
   - What we know: expo-google-fonts repo includes JetBrains Mono. Bundling .ttf files also works.
   - What's unclear: Whether the expo-google-fonts package has the exact weights we need (Regular, Medium, Bold).
   - Recommendation: Bundle .ttf files from official JetBrains site for guaranteed control. Fallback to `@expo-google-fonts/jetbrains-mono` if bundling causes issues.

3. **Theme extraction automation**
   - What we know: The web CSS has 13 theme definitions with all color tokens as CSS custom properties.
   - What's unclear: Whether border-subtle opacity values and shadow definitions differ per theme in ways that are hard to extract programmatically.
   - Recommendation: Start with manual extraction for Mocha (default), then build the extraction script. Verify Latte (light) and one custom theme (Nord or Cherry) as the second and third themes.

## Validation Architecture

### Test Framework

| Property | Value |
|----------|-------|
| Framework | Maestro (latest CLI) + Jest 29.x |
| Config file | maestro/flows/*.yaml (Maestro), jest.config.js (Jest, Expo default) |
| Quick run command | `maestro test maestro/flows/app-launch.yaml` |
| Full suite command | `maestro test maestro/flows/` |

### Phase Requirements to Test Map

| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| FOUND-01 | iOS dev build launches | smoke | `maestro test maestro/flows/app-launch.yaml` (on iOS Simulator) | Wave 0 |
| FOUND-02 | Android dev build launches | smoke | `maestro test maestro/flows/app-launch.yaml` (on Android Emulator) | Wave 0 |
| FOUND-03 | Tab navigation works, all routes accessible | e2e | `maestro test maestro/flows/app-launch.yaml` (tap each tab, assertVisible) | Wave 0 |
| FOUND-04 | Mocha theme renders on first frame, no flash | manual-only | Visual inspection of cold launch recording; Maestro screenshot comparison | N/A (manual) |
| FOUND-05 | 13 themes match web CSS | screenshot-comparison | `node scripts/verify-themes.js` (compare mobile screenshots to web) | Wave 0 |
| FOUND-06 | Custom fonts render before first frame | manual-only | Visual inspection; Maestro screenshot shows correct font on first capture | N/A (manual) |
| FOUND-07 | Component library renders correctly | unit + storybook | `npx jest --testPathPattern=components` | Wave 0 |
| FOUND-08 | Maestro runs on iOS Simulator with screenshot | smoke | `maestro test maestro/flows/app-launch.yaml && ls maestro/screenshots/` | Wave 0 |

### Sampling Rate

- **Per task commit:** `npx jest --bail --testPathPattern=<changed-module>`
- **Per wave merge:** `npx jest && maestro test maestro/flows/app-launch.yaml`
- **Phase gate:** Full Jest suite green + Maestro app-launch flow passes with screenshots on iOS Simulator

### Wave 0 Gaps

- [ ] `maestro/flows/app-launch.yaml` - basic launch + tab navigation + screenshot flow
- [ ] `jest.config.js` - Expo default (generated by create-expo-app)
- [ ] `scripts/verify-themes.js` - compare theme token values against web CSS
- [ ] `scripts/extract-themes.js` - parse styles.css and generate tokens.ts
- [ ] Maestro CLI + idb-companion installed on Mac Mini
- [ ] iOS Simulator build profile in eas.json

## Sources

### Primary (HIGH confidence)
- Web CSS source of truth: `src/web/public/styles.css` lines 20-80 (Mocha), lines 7457-8350 (all other themes)
- [Expo SDK 54 Changelog](https://expo.dev/changelog/sdk-54) - SDK version, New Architecture default, build features
- [Expo create-expo-app docs](https://docs.expo.dev/more/create-expo/) - tabs template, project initialization
- [expo-router typed routes](https://docs.expo.dev/router/reference/typed-routes/) - experiments.typedRoutes config
- [expo-font docs](https://docs.expo.dev/versions/latest/sdk/font/) - useFonts hook, SplashScreen pattern
- [Expo SplashScreen docs](https://docs.expo.dev/versions/latest/sdk/splash-screen/) - preventAutoHideAsync API
- [react-native-mmkv Zustand wrapper](https://github.com/mrousavy/react-native-mmkv/blob/main/docs/WRAPPER_ZUSTAND_PERSIST_MIDDLEWARE.md) - official MMKV-Zustand integration docs
- [Maestro React Native docs](https://docs.maestro.dev/get-started/supported-platform/react-native) - setup, YAML flows, screenshot capture
- [Expo push notifications setup](https://docs.expo.dev/push-notifications/push-notifications-setup/) - dev build requirement

### Secondary (MEDIUM confidence)
- [zustand-mmkv-storage npm](https://www.npmjs.com/package/zustand-mmkv-storage) - third-party adapter, verified against official MMKV docs
- [@expo-google-fonts/plus-jakarta-sans npm](https://www.npmjs.com/package/@expo-google-fonts/plus-jakarta-sans) - font package availability
- [Maestro Expo dev builds blog](https://maestro.dev/blog/running-maestro-ui-tests-in-an-expo-development-builds) - iOS Simulator testing workflow
- [Expo EAS Workflows Maestro](https://docs.expo.dev/eas/workflows/examples/e2e-tests/) - CI integration for Maestro

### Tertiary (LOW confidence)
- Maestro + macOS Sequoia compatibility: [GitHub issue #173](https://github.com/mobile-dev-inc/maestro-docs/issues/173) - known issue, unclear if resolved
- SplashScreen.preventAutoHideAsync race conditions: [GitHub issue #31875](https://github.com/expo/expo/issues/31875) - reported but may be user error

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH - all libraries verified against Expo SDK 54, versions confirmed
- Architecture: HIGH - patterns from official docs and verified community standards
- Theme system: HIGH - web CSS is the definitive source; MMKV sync pattern is well-documented
- Pitfalls: HIGH - confirmed via GitHub issues, official docs, and community reports
- Maestro setup: MEDIUM - macOS Sequoia compatibility is uncertain; basic setup is well-documented

**Research date:** 2026-03-28
**Valid until:** 2026-04-28 (30 days; Expo SDK stable, no major releases expected)

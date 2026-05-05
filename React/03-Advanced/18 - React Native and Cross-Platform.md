---
tags: [react, advanced, architecture]
aliases: [React Native, Expo, Cross-Platform]
level: Advanced
---

# React Native and Cross-Platform

> **One-liner**: **React Native (RN)** is React with a different renderer that produces native iOS/Android UI (not a webview); **Expo** is the batteries-included framework around RN; you share components/business logic with web — but the platform layer is genuinely different.

---

## Quick Reference

| Concept | Web (DOM) | React Native |
|---------|-----------|--------------|
| Render target | DOM (`react-dom`) | iOS/Android views (`react-native`) |
| Layout | CSS / flexbox | Flexbox only (no display:block, etc.) |
| Styling | CSS / Tailwind | `StyleSheet.create` (subset of CSS, camelCase) |
| Tags | `div`, `span`, `img` | `<View>`, `<Text>`, `<Image>` |
| Routing | React Router / Next | **Expo Router** (file-system) or React Navigation |
| Forms | `<input>` | `<TextInput>` |
| Network | `fetch` | `fetch` |
| Storage | `localStorage` | `AsyncStorage`, MMKV |
| Build | Vite/Webpack | Metro bundler |
| Run | Browser | iOS Simulator, Android Emulator, real device |
| Deploy | CDN | App Store / Play Store (or Expo OTA) |
| New Architecture | n/a | **Fabric** + **TurboModules** + **JSI** (default in 2025) |

---

## Core Concept

React Native swaps out `react-dom` for a renderer that talks to iOS / Android's native UI toolkits. Your component model — JSX, hooks, state, effects — is identical. What changes is the **primitive elements** and **APIs available to you**.

You don't get HTML elements. There's no `<div>` — you write `<View>`. There's no `text-overflow: ellipsis` — you set props on `<Text>`. CSS is gone, replaced by JS objects with a CSS-flexbox subset. The good news: this is a small adjustment, and Tailwind-style RN libraries (NativeWind) bring a familiar styling API.

**Expo** is the standard framework around RN. It gives you: file-system routing (Expo Router, very Next-like), built-in modules (camera, location, secure storage), OTA updates (push JS bundle without app store), build service (no need to maintain Xcode/Android Studio yourself), and dev tooling. **In 2025 most new RN projects start with Expo.**

What to share between web and native:
- ✅ Pure logic (hooks, utilities, schemas, API clients).
- ✅ Some component logic (extract presentational into platform-specific files).
- ❌ Layout/styling (different APIs).
- ❌ Navigation (different abstractions).
- ❌ Platform-specific UX (don't make iOS look like web).

The "build once, deploy everywhere" pitch is half-true. Share **logic**, not look-and-feel.

---

## Syntax & API

### Expo project setup

```bash
npx create-expo-app@latest my-app
cd my-app
npx expo start                  # opens dev server, scan QR with Expo Go app
```

### A screen — note primitive components, no CSS

```tsx
// app/index.tsx (Expo Router)
import { useState } from "react";
import { View, Text, Pressable, StyleSheet } from "react-native";

export default function Home() {
  const [count, setCount] = useState(0);

  return (
    <View style={styles.container}>
      <Text style={styles.title}>{count}</Text>
      <Pressable onPress={() => setCount(c => c + 1)} style={styles.btn}>
        <Text style={styles.btnText}>+1</Text>
      </Pressable>
    </View>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, alignItems: "center", justifyContent: "center", gap: 16 },
  title:     { fontSize: 32, fontWeight: "bold" },
  btn:       { backgroundColor: "#2563eb", paddingHorizontal: 24, paddingVertical: 12, borderRadius: 8 },
  btnText:   { color: "white", fontWeight: "600" },
});
```

### Expo Router — file-system routing

```
app/
├── _layout.tsx          # root stack
├── index.tsx            # /
├── settings.tsx         # /settings
└── users/
    └── [id].tsx         # /users/:id
```

```tsx
// app/users/[id].tsx
import { useLocalSearchParams } from "expo-router";

export default function User() {
  const { id } = useLocalSearchParams<{ id: string }>();
  return <Text>User {id}</Text>;
}
```

### Native modules — camera, location

```bash
npx expo install expo-camera expo-location
```

```tsx
import { useCameraPermissions, CameraView } from "expo-camera";

function Scanner() {
  const [perm, requestPerm] = useCameraPermissions();

  if (!perm) return null;
  if (!perm.granted) return <Pressable onPress={requestPerm}><Text>Grant camera</Text></Pressable>;

  return <CameraView style={{ flex: 1 }} />;
}
```

### Sharing logic via a monorepo

```
packages/
├── ui-web/       # web-specific components
├── ui-native/    # RN components
├── core/         # shared: hooks, schemas, API client, types
└── ...
apps/
├── web/          # Vite + React + ui-web + core
└── mobile/       # Expo + ui-native + core
```

```ts
// packages/core/hooks/useUser.ts
export function useUser(id: string) {
  // works on both web and native (uses fetch + TanStack Query)
}
```

### Platform-specific files (`.ios.tsx`, `.android.tsx`, `.web.tsx`)

```tsx
// Button.tsx          ← shared (rare)
// Button.ios.tsx      ← iOS-specific
// Button.android.tsx  ← Android-specific
// Button.web.tsx      ← web-specific (with react-native-web)
// Bundler picks the right file per target.
```

### `react-native-web` — run RN code in the browser

```bash
npm install react-native-web react-dom
# Configure Vite/Webpack alias: "react-native" → "react-native-web"
```

Useful for sharing the same component tree between RN and web — at the cost of some web-feature compromises.

---

## Common Patterns

```tsx
// Pattern: NativeWind for Tailwind-in-RN
// className="flex-1 items-center justify-center bg-white"
import { View, Text } from "react-native";
<View className="flex-1 items-center justify-center bg-white">
  <Text className="text-3xl font-bold">Hello</Text>
</View>
```

```tsx
// Pattern: shared TanStack Query — same hook, same API, different platform
const { data, isLoading } = useQuery({ queryKey: ["user", id], queryFn: () => fetchUser(id) });
```

---

## Gotchas & Tips

- **No CSS, no `<div>`, no `<a>`.** Every primitive is `<View>`/`<Text>`/`<Image>`/`<ScrollView>`/etc. Internalize this early.
- **Text MUST be inside `<Text>`.** A bare string in `<View>` crashes.
- **Flexbox `flexDirection` defaults to `column`** in RN (vs `row` on web). Easy to forget.
- **No `position: fixed`.** Use absolute positioning + insets.
- **Touch targets**: minimum 44×44 pt on iOS. Tiny buttons frustrate.
- **Performance**: lots of mounted views are slow. Use `FlatList`/`SectionList` for long lists (built-in virtualization).
- **Animations**: `Animated` API is built-in but verbose; **Reanimated 3** runs animations on the UI thread (60–120fps even when JS is busy).
- **App Store / Play Store** review delays releases. **Expo OTA updates** push JS bundle changes without store review (within rules).
- **New Architecture (Fabric + TurboModules)** is on by default in 2025 — much faster, but some legacy native modules need updates.
- **Don't try to share UI verbatim with web.** Native users expect native gestures, transitions, fonts. Share *logic*; rebuild *presentation*.
- **For "I just need a web view in an app"**, RN is overkill — use Capacitor / a PWA.

---

## See Also

- [[10 - useEffect Basics]]
- [[06 - Custom Hooks]]
- [[09 - Styling]]
- [[12 - State Management]]

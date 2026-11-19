Mobile — React Native + Expo
============================

> Read **[General Principles](/Coding-Standards/General-Principles)** first — this page only covers what's specific to our React Native stack. Much of the [Frontend — React + TypeScript](/Coding-Standards/Frontend-React-TypeScript) page applies here too: same query keys, same Zod-first schemas, same server-vs-client state split. This page covers what's *different* because it's a phone.
> **Which tools we use, why, and how to set them up:** [Tech Stack → Mobile Stack](/Tech-Stack/Mobile-Stack). This page is how to write code with them day to day.

* * *

## At a Glance

| Topic | The rule |
| --- | --- |
| [Folder structure](#folder-structure) | **Routes are thin, screens hold the UI**<br/><sub>`src/app/` only reads params and renders a screen from `src/screens/`.</sub> |
| [Long lists](#long-lists-use-flatlist) | **`FlatList` for anything driven by an array**<br/><sub>`.map()` in a `ScrollView` mounts every row at once and drops frames.</sub> |
| [Re-renders](#keep-references-stable-across-renders) | **Memoize list rows, style with `className`**<br/><sub>Inline objects and arrow functions re-render every row on every render.</sub> |
| [State](#state-follows-the-web-rules) | **API data in TanStack Query, UI state in Zustand**<br/><sub>Same rule as the web. Screens only call hooks, never API functions directly.</sub> |
| [Storage](#storing-tokens-and-data) | **Tokens in SecureStore, everything else in AsyncStorage**<br/><sub>AsyncStorage is an unencrypted file, readable on a rooted device.</sub> |
| [Calling the API](#calling-the-api) | **Always use the shared `apiClient`**<br/><sub>It attaches the token and logs out cleanly on a 401, so call sites don't have to.</sub> |
| [Permissions](#every-native-capability-has-a-denied-path) | **Handle "no" and "cancel" on every native capability**<br/><sub>Denied is a normal state. Ignoring it looks like a broken app.</sub> |
| [Layout](#safe-areas-and-platform-differences) | **Ask the device for safe areas**<br/><sub>Hard-coded notch padding is wrong on every other phone.</sub> |
| [Native config and packages](#native-config-and-packages) | **`npx expo install`, config in `app.config.ts`, never edit `ios/` or `android/`**<br/><sub>Prebuild regenerates the native folders and erases hand edits.</sub> |
| [Images and logging](#remote-images) | **`expo-image` for remote images, a logger instead of `console.log`**<br/><sub>Cached images, and no debug output in shipped builds.</sub> |
| [Linting and formatting](#linting-and-formatting) | **Expo's ESLint config + Prettier, `tsc --noEmit` before every commit**<br/><sub>There's no build step to catch a type error before a user does.</sub> |
| [Testing](#testing) | **Start with pure helpers, then a few critical flows**<br/><sub>Jest + RNTL for components, Maestro for flows on a real device.</sub> |

* * *

## What makes mobile different

Four constraints that don't exist on the web, and that most of this page follows from.

| Constraint | What it means for your code |
| --- | --- |
| **Two threads** | Your JS runs on one thread, the UI is drawn on another. Block the JS thread and the app doesn't just feel slow — animations stutter and taps don't register. |
| **The device is weaker than your laptop** | A mid-range Android phone is the test, not your simulator. A list that's fine with 20 rows on a MacBook drops frames at 200 rows on a real device. |
| **Nothing is guaranteed** | The network drops in a lift. The app gets backgrounded mid-upload. Permission is denied. All three are normal states to handle, not edge cases to ignore. |
| **You can't hotfix natively** | A native change means a new store build and review. Get native config right the first time. |

* * *

## Concepts you'll meet on this codebase

Short definitions of the terms the rest of the page uses. Skip the ones you already know.

### Route file and screen component

Expo Router turns files in `src/app/` into screens. A **route file** is plumbing: read the params, render the screen. The **screen component** in `src/screens/` holds the actual UI and logic. Keeping them separate is what makes a screen reusable, testable, and openable from more than one route.

```tsx
// ✅ src/app/(services)/appraisal/[id].tsx — the whole route file
import { useLocalSearchParams } from 'expo-router';
import { AppraisalDetailScreen } from '@/screens/appraisal/AppraisalDetailScreen';

export default function AppraisalDetailRoute() {
  const { id } = useLocalSearchParams<{ id: string }>();
  return <AppraisalDetailScreen appraisalId={id} />;
}
```

### Route groups

A folder in parentheses — `(auth)`, `(tabs)`, `(services)` — groups screens under a shared `_layout.tsx` without adding a segment to the path. That's how the logged-out screens, the tab bar, and the flows pushed on top of it stay separated.

### The two storage tiers

`expo-secure-store` is encrypted by the OS (iOS Keychain / Android Keystore). `AsyncStorage` is a plain unencrypted file. Tokens and credentials go in the first; cached data and preferences go in the second. This is not a preference — see [Storing tokens and data](#storing-tokens-and-data).

### Reanimated worklets

Reanimated runs animation code on the UI thread, in a separate JS context, so it keeps running even when your app's main thread is busy. That's why animated values are `useSharedValue` rather than `useState`, and why you can't reach into normal component state from inside an animation.

### Continuous Native Generation

`ios/` and `android/` are **generated** from the app config (`app.config.ts`) and config plugins. Hand-editing them works until the next `npx expo prebuild`, which silently throws your change away. Native config belongs in the app config or a config plugin. See [Native config and packages](#native-config-and-packages).

* * *

## Folder structure

```
src/
├─ app/           → routes only. Thin files, params in, screen out.
│  ├─ _layout.tsx →   root: providers, splash, session handling
│  ├─ (auth)/     →   login, OTP, password reset (logged out)
│  ├─ (tabs)/     →   tab bar and its screens
│  ├─ (services)/ →   feature flows pushed on top of the tabs
│  └─ settings/
├─ screens/       → one folder per feature flow, the real UI lives here
├─ components/    → shared UI, grouped by kind: ui/ forms/ modals/ cards/
├─ hooks/         → query/mutation hooks, grouped by domain: appraisals/ appointments/
├─ api/           → one file per backend domain, plain async functions
├─ stores/        → Zustand stores
├─ lib/           → apiClient, queryClient, queryKeys, logger, native wrappers
├─ types/         → shared types, API response shapes
├─ utils/         → pure helpers — formatting, mapping
├─ constants/     → colors, enums, static option lists
└─ config/        → runtime config read from expo-constants (runtimeConfig.ts)
```

Two axes, on purpose: **components group by kind** (they're reused across features), **hooks group by domain** (they belong to one).

> ⚠️ Don't let `src/app/` grow real logic. A 400-line route file can't be tested, reused, or opened from a deep link and a tab. See [Route file and screen component](#route-file-and-screen-component).

* * *

## Long lists use FlatList

The single biggest performance mistake in a React Native codebase. `ScrollView` renders **every** child immediately; `FlatList` renders only what's on screen.

```tsx
// ❌ 300 rows = 300 mounted components before the screen shows anything
<ScrollView>
  {appraisals.map((a) => <AppraisalCard key={a.id} appraisal={a} />)}
</ScrollView>

// ✅ Renders ~10, recycles as you scroll
<FlatList
  data={appraisals}
  keyExtractor={(a) => String(a.id)}
  renderItem={({ item }) => <AppraisalCard appraisal={item} />}
  onEndReached={() => hasNextPage && fetchNextPage()}
  onEndReachedThreshold={0.5}
/>
```

`ScrollView` is fine for a form or a detail screen — a known, small, fixed set of children. Anything driven by an array from the API is a `FlatList`.

> 💡 **Pair it with `useInfiniteQuery`.** Query owns the cursor, the pages and the loading flag; `onEndReached` just asks for the next page. Don't hand-roll page-number state.

* * *

## Keep references stable across renders

Every re-render rebuilds inline objects and arrow functions, which makes every row in a list re-render with it.

```tsx
// ❌ New function and new style object per row, per render
<FlatList
  data={items}
  renderItem={({ item }) => (
    <Pressable style={{ padding: 16 }} onPress={() => onSelect(item.id)}>
      <Text>{item.name}</Text>
    </Pressable>
  )}
/>

// ✅ Stable references; the row memoizes properly
const AppraisalRow = memo(function AppraisalRow({ item, onSelect }: Props) {
  const handlePress = useCallback(() => onSelect(item.id), [item.id, onSelect]);
  return (
    <Pressable className="p-4" onPress={handlePress}>
      <Text>{item.name}</Text>
    </Pressable>
  );
});
```

Use NativeWind's `className` rather than inline `style` objects — it compiles to a stable style reference at build time, so it doesn't have this problem in the first place.

> 📝 Don't scatter `memo`/`useCallback` everywhere by reflex — that's the premature optimisation [General Principles → Keep it simple](/Coding-Standards/General-Principles#keep-it-simple) warns about. Apply it to list rows and expensive subtrees, which is where it actually pays.

* * *

## State follows the web rules

Server data belongs to **TanStack Query**, UI state belongs to **Zustand** or local `useState`. The reasoning and the ❌/✅ examples are on the [Frontend page](/Coding-Standards/Frontend-React-TypeScript#keep-server-data-out-of-stores) — they apply here unchanged.

Three things mobile adds:

**One key factory for the whole app**, in `src/lib/queryKeys.ts`:

```ts
// ✅ src/lib/queryKeys.ts
export const queryKeys = {
  appraisals: {
    all: ['appraisals'] as const,
    infinite: (search = '', status = 'all') => [...queryKeys.appraisals.all, { search, status }] as const,
    detail: (id: string | number) => [...queryKeys.appraisals.all, 'detail', id] as const,
  },
};
```

**API function → query hook → screen.** `src/api/` holds plain async functions that only know request shapes; `src/hooks/` wraps them in TanStack Query and owns caching. Screens only call hooks.

```ts
// src/api/appraisal.ts — request shape only
export const getAppraisal = (id: string) =>
  apiClient.get<Appraisal>(`/appraisals/${id}`).then((r) => r.data);

// src/hooks/appraisals/useAppraisal.ts — caching
export const useAppraisal = (id: string) =>
  useQuery({ queryKey: queryKeys.appraisals.detail(id), queryFn: () => getAppraisal(id) });
```

**A screen isn't unmounted when you navigate away.** Expo Router keeps screens in the stack alive. A `useEffect` that assumes "mounts once when I open this screen" will not behave the way you expect; use `useFocusEffect` when you mean "when this screen becomes visible".

```tsx
// ❌ Runs once. Come back to the screen later and the list is stale.
useEffect(() => {
  refetch();
}, []);

// ✅ Runs every time the screen comes into view
useFocusEffect(
  useCallback(() => {
    refetch();
  }, [refetch]),
);
```

> 💡 **Persist deliberately.** Zustand's `persist` middleware is how state survives the app being killed. Choose the storage backend per store based on sensitivity — never by default. See [Storing tokens and data](#storing-tokens-and-data).

* * *

## Storing tokens and data

Pick the storage by **how sensitive the data is**, every time.

| Data | Store | Why |
| --- | --- | --- |
| Access and refresh tokens, credentials | **`expo-secure-store`** | Encrypted by the OS |
| Cached lists, preferences, onboarding flags | **`AsyncStorage`** | Fast, but a plain unencrypted file |

```ts
// ✅ token: encrypted by the OS
await SecureStore.setItemAsync('accessToken', token);

// ✅ preference: not sensitive, plain storage is fine
await AsyncStorage.setItem('recentSearches', JSON.stringify(searches));

// ❌ token in a plain file: readable on a rooted or jailbroken device
await AsyncStorage.setItem('accessToken', token);
```

The same choice applies when a Zustand store is persisted: the session store uses the SecureStore adapter, everything else uses AsyncStorage. The adapter setup is in [Tech Stack → Storage](/Tech-Stack/Mobile-Stack#storage).

* * *

## Calling the API

There is one shared Axios instance, `apiClient` in `src/lib/`. Its interceptors attach the token to every request and log the user out once when the session expires (a 401). Feature code always goes through it. How it's set up is in [Tech Stack → HTTP Client](/Tech-Stack/Mobile-Stack#http-client).

```ts
// ✅ src/api/appraisal.ts: token and 401 handling are already taken care of
import { apiClient } from '@/lib/apiClient';

export const getAppraisal = (id: string) =>
  apiClient.get<Appraisal>(`/appraisals/${id}`).then((r) => r.data);

// ❌ token attached by hand: forgotten at the next call site,
//    and an expired session never logs the user out
const token = await SecureStore.getItemAsync('accessToken');
const res = await axios.get(`${API_URL}/appraisals/${id}`, {
  headers: { Authorization: `Bearer ${token}` },
});
```

* * *

## Every native capability has a denied path

Camera, photos, location, notifications, calendar — the user can say no, and can revoke it later from system settings. Denied is a normal state.

```tsx
// ❌ Assumes yes. On denial: nothing happens, no message, user thinks the app is broken.
const openCamera = async () => {
  await ImagePicker.requestCameraPermissionsAsync();
  const result = await ImagePicker.launchCameraAsync();
  setPhoto(result.assets[0].uri);
};

// ✅ Handle the no
const openCamera = async () => {
  const { granted, canAskAgain } = await ImagePicker.requestCameraPermissionsAsync();

  if (!granted) {
    return canAskAgain
      ? showMessage('Camera access is needed to photograph the vehicle.')
      : promptOpenSettings('Enable camera access in Settings to continue.');
  }

  const result = await ImagePicker.launchCameraAsync();
  if (result.canceled) return;
  setPhoto(result.assets[0].uri);
};
```

> ⚠️ Note `result.canceled` — the user backing out of the picker is also a normal path, and reading `assets[0]` without checking it is a crash.

Keep permission requests behind one hook (`usePermissions`) rather than repeating this at every call site, and write the usage description in the app config — both stores reject builds that request a capability without explaining why. See [Native config and packages](#native-config-and-packages).

* * *

## Safe areas and platform differences

```tsx
// ❌ Hardcoded padding for "the notch" — wrong on every other device
<View style={{ paddingTop: 44 }}>

// ✅ Ask the device
const insets = useSafeAreaInsets();
<View style={{ paddingTop: insets.top }}>
```

`Platform.OS` checks are fine where the platforms genuinely differ (keyboard behaviour, elevation vs. shadow, back-button handling). They're a smell when they're compensating for a layout you hardcoded.

* * *

## Native config and packages

Anything that touches native code follows Expo's rules, because `ios/` and `android/` are regenerated from config (see [Continuous Native Generation](#continuous-native-generation)).

**Install with `npx expo install`.** It picks the package version that matches the project's Expo SDK. A plain `yarn add` can pull a version whose native code doesn't match, which builds fine and crashes at runtime.

```bash
# ✅ version matched to the Expo SDK
npx expo install expo-camera

# ❌ for anything expo-* or with native code
yarn add expo-camera
```

**Native settings go in `app.config.ts`, not in `ios/` or `android/`.** Permission messages are a common example:

```ts
// ✅ app.config.ts: survives every prebuild
plugins: [
  ['expo-camera', { cameraPermission: 'Allow $(PRODUCT_NAME) to photograph the vehicle.' }],
],

// ❌ editing ios/<App>/Info.plist by hand: erased by the next npx expo prebuild
```

**Reanimated's Babel plugin goes last.** If it isn't, animations silently don't run.

```js
// babel.config.js
plugins: [
  // ...other plugins
  'react-native-reanimated/plugin', // must stay LAST
],
```

**`extra` is for config, never secrets.** Everything in `app.config.ts` → `extra` ships inside the app. Read it through `runtimeConfig`, not `Constants` directly.

```ts
// ✅ app.config.ts: a public API base URL is fine
extra: { API_BASE_URL: process.env.API_BASE_URL },

// ❌ anyone who downloads the app can read this
extra: { STRIPE_SECRET_KEY: process.env.STRIPE_SECRET_KEY },
```

The full config flow is in [Tech Stack → Configuration](/Tech-Stack/Mobile-Stack#configuration).

* * *

## Remote images

Use `expo-image` for anything loaded from a URL. It caches to disk, so a list of photos doesn't re-download every time the user scrolls back.

```tsx
// ✅ cached, with a short fade-in
import { Image } from 'expo-image';

<Image source={{ uri: vehicle.photoUrl }} style={{ width: 120, height: 90 }} contentFit="cover" transition={200} />

// ❌ no disk cache: the same photo downloads again and again
import { Image } from 'react-native';
```

* * *

## Logging

Log through one small wrapper in `src/lib/logger.ts` that does nothing in production builds. A stray `console.log` in shipped code slows the JS thread and can leak data to the device logs.

```ts
// ✅ src/lib/logger.ts
export const logger = {
  debug: (...args: unknown[]) => {
    if (__DEV__) console.log(...args);
  },
  error: (...args: unknown[]) => {
    if (__DEV__) console.error(...args);
  },
};

// usage
logger.debug('appraisal loaded', appraisal.id);

// ❌ ships to every user's phone
console.log('appraisal loaded', appraisal);
```

* * *

## Do / Don't

Each row links to the section with its full example.

| ✅ Do | ❌ Don't | Example |
| --- | --- | --- |
| Tokens and credentials in `expo-secure-store` | Any credential in `AsyncStorage` — it's readable on a rooted device | [Storage](#storing-tokens-and-data) |
| `FlatList` for anything array-driven | `.map()` inside a `ScrollView` | [Long lists](#long-lists-use-flatlist) |
| One shared Axios instance with auth + 401 interceptors | Attach the token by hand at each call site | [Calling the API](#calling-the-api) |
| `npx expo install <pkg>` | `yarn add <pkg>` for anything `expo-*` or with native code | [Native config](#native-config-and-packages) |
| Native config in `app.config.ts` / config plugins | Hand-edit `ios/` or `android/` — prebuild will erase it | [Native config](#native-config-and-packages) |
| Reanimated's Babel plugin **last** in `babel.config.js` | Wonder for an afternoon why animations silently don't run | [Native config](#native-config-and-packages) |
| `expo-image` for remote images | `Image` from `react-native` for anything you want cached | [Remote images](#remote-images) |
| A logger wrapper that no-ops in production | `console.log` left in shipped code | [Logging](#logging) |
| API base URLs in `app.config.ts` → `extra` | Any secret in `extra` — it ships inside the bundle and is readable | [Native config](#native-config-and-packages) |
| `useSafeAreaInsets()` | Hardcoded status-bar padding | [Safe areas](#safe-areas-and-platform-differences) |
| Handle denied and cancelled on every permission | Assume the user said yes | [Permissions](#every-native-capability-has-a-denied-path) |
| `useFocusEffect` for "when this screen is visible" | A `useEffect` that assumes the screen remounts each time it's opened | [State](#state-follows-the-web-rules) |
| Test on a real mid-range Android device | Ship on "works on my simulator" | [What makes mobile different](#what-makes-mobile-different) |

* * *

## Linting and formatting

Same tools as the [Frontend page](/Coding-Standards/Frontend-React-TypeScript#linting-and-formatting) — Prettier + ESLint — with one shortcut: Expo projects can bootstrap the config instead of assembling it by hand.

```bash
npx expo lint
```

This installs ESLint with `eslint-config-expo` already wired up and adds a `lint` script to `package.json`. Add Prettier on top with the same [`.prettierrc` as the Frontend page](/Coding-Standards/Frontend-React-TypeScript#linting-and-formatting) — one style across web and mobile means a developer moving between them isn't relearning quote and comma conventions.

Then put Prettier last in the ESLint config so the two never conflict:

```js
// eslint.config.js
const expoConfig = require('eslint-config-expo/flat');
const prettier = require('eslint-config-prettier');
const { defineConfig } = require('eslint/config');

module.exports = defineConfig([expoConfig, prettier]);
```

- Pre-commit: `husky` + `lint-staged`, same as web — format and lint only staged files, plus a **project-wide** typecheck:

```js
// lint-staged.config.js
module.exports = {
  '*.{ts,tsx}': ['prettier --write', 'eslint --fix', () => 'tsc --noEmit'],
  '*.{json,md}': 'prettier --write',
};
```

- CI: `yarn lint`, `npx prettier --check .`, and `tsc --noEmit` on every PR. This stack has no build step to catch a type error before a user does, which is what makes `tsc --noEmit` non-negotiable here rather than a nice-to-have.

* * *

## Testing

Automated testing isn't yet a day-one requirement on our mobile projects — see the note in the [Mobile Stack](/Tech-Stack/Mobile-Stack). Where a project does invest:

- **Jest + `jest-expo` + React Native Testing Library** for units and components — assert on what the user sees (`getByText`, `getByRole`), not on internal state.
- **Maestro** for a handful of critical flows on a real device or emulator (login, submit), not every screen.
- Pure helpers in `utils/` are the cheapest thing to test and usually where the business rules live — start there.

```ts
// src/utils/formatMileage.test.ts — a pure helper, no device or mocks needed
import { formatMileage } from './formatMileage';

test('adds thousands separators and the unit', () => {
  expect(formatMileage(125000)).toBe('125,000 km');
});
```

> ⚠️ What's non-negotiable regardless: `tsc --noEmit` and lint pass before commit. React Native has no build step that would otherwise catch a type error before a user does.

* * *

## Further reading

- [Expo — Best practices](https://docs.expo.dev/develop/development-builds/introduction/)
- [React Native — Performance overview](https://reactnative.dev/docs/performance)
- [Optimizing FlatList configuration](https://reactnative.dev/docs/optimizing-flatlist-configuration)
- [TanStack Query — React Native](https://tanstack.com/query/latest/docs/framework/react/react-native)

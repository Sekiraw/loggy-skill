# Setting up loggy-react-native in a React Native or Expo app

loggy-react-native works with React Native 0.76 or later and React 19, with or without Expo. It has no native code, so it runs in Expo Go and needs no new build. Check `package.json` for `react-native` and `expo`, and whether the app uses Expo Router (`app/_layout.tsx` or `src/app/_layout.tsx`).

## 1. Install

In an Expo app:

```bash
npx expo install loggy-react-native expo-sqlite
```

Without Expo, use the package manager the lockfile shows, and leave out expo-sqlite (see step 3).

## 2. The DSN

Get it as for a Next.js app: from `create_project`, `list_projects` or the project's Settings page. Use a separate project for the app, not the website's. Put it in `.env`:

```bash
EXPO_PUBLIC_LOGGY_DSN=https://lgy_yourprojectkey@loggy-next.vercel.app
```

Expo doesn't put environment variables into packages, so the app passes it to the component itself. For EAS builds, the variable also has to be set in the EAS environment (`eas env:create`, or the project's Environment variables on expo.dev). The DSN isn't a secret: phones send logs with it.

## 3. The component

Add `<Loggy />` once, in the root layout, before the navigator:

```tsx
// app/_layout.tsx
import "expo-sqlite/localStorage/install";
import { Stack, usePathname } from "expo-router";
import { Loggy } from "loggy-react-native";

export default function RootLayout() {
  const pathname = usePathname();
  return (
    <>
      <Loggy dsn={process.env.EXPO_PUBLIC_LOGGY_DSN} screen={pathname} />
      <Stack />
    </>
  );
}
```

- **The first line** gives the phone a synchronous localStorage. A crash ends the app before its report can be sent, so the package writes errors there and sends them on the next launch. It also keeps the last flag values between launches. Without Expo, any synchronous localStorage on `globalThis` works, such as one backed by MMKV. Without one, everything else still works.
- **`screen`** puts the screen on each log as its path. With React Navigation instead of Expo Router, pass the current route's name.
- **Who is signed in:** if the root layout can see the user, add `context={user ? { key: user.id } : {}}`. Flags then follow them, and segments can target them. `{}` when signed out, so flags stop following the previous user.

## What gets captured

- Uncaught errors, including the ones that end the app, marked `data.fatal: true`. In a production build the package waits up to a second for the report to leave before React Native closes the app.
- Unhandled promise rejections in production builds. In development React Native shows them itself, and they arrive through the console.
- Render errors React reports, including those an error boundary caught, with `data.componentStack`.
- `console.warn` and `console.error`. `console.log` and `console.info` only with `consoleLevel="info"` or lower.
- The release: the app version from its Expo config, plus the EAS Update id when one runs (`1.4.0+3f2a1b9c`). The environment is `development` in a development build (`__DEV__`) and `production` otherwise; pass `environment="preview"` for preview builds.
- The device and app version, as the log's user agent: `Parkee/1.4.0 (Android 14; Google Pixel 8)`.
- A session per launch, and a new one after 30 minutes in the background.

Logs are sent every two seconds and when the app goes to the background.

## Things the app has to do itself

- **Failed requests.** Data libraries such as React Query catch errors and don't print them, so nothing reaches Loggy on its own. Report them from the library's global error handler:

  ```ts
  import { log } from "loggy-react-native";
  import { MutationCache, QueryCache, QueryClient } from "@tanstack/react-query";

  export const queryClient = new QueryClient({
    queryCache: new QueryCache({ onError: (error, query) => log.error(`Query failed: ${query.queryKey.join("/")}`, error) }),
    mutationCache: new MutationCache({ onError: (error) => log.error("Change failed", error) }),
  });
  ```

- **Business events** worth searching: `log.info("Parking started", { zone, minutes })`, `log.warn("Top-up failed", error)`. `log.*` is sent whatever `consoleLevel` says, and prints to the console too.

## Flags

The same API as loggy-next's browser side: `useFlag("new-map", false)` in a component, `getFlag` and `getFlags` elsewhere, `identify(context)` after signing in if `context` isn't passed to `<Loggy />`. Values are checked every 30 seconds while the app is open and when it returns to the foreground.

## Check it works

1. Run the app (`npx expo start`) and trigger a `console.warn`.
2. Call `search_logs` with `query: "source:mobile"` and `range: "15m"` on the project.
3. If nothing arrives: check that `EXPO_PUBLIC_LOGGY_DSN` is set and the dev server was restarted after `.env` changed. A DSN with `localhost` points at the phone itself; use the computer's IP address there.

## Not yet

- Production stack traces point into the Hermes bundle and aren't mapped to source files.
- No page speed, request logs or route list: those are Next.js features.

# Setting up loggy-next in a Next.js app

loggy-next works with the App Router on Next.js 15 or later and React 19. Check `package.json` and the `app/` folder first. With the Pages Router only, it captures browser logs and nothing else, and it isn't tested there.

## 1. Install

Use the package manager the lockfile shows:

```bash
npm install loggy-next     # or: pnpm add loggy-next / yarn add loggy-next / bun add loggy-next
```

## 2. The DSN

The DSN tells the app which project its logs belong to. It looks like `https://lgy_<publickey>@loggy-next.vercel.app`.

- With the change tools, create a project for the app with `create_project`. It returns the DSN. Otherwise take it from `list_projects`, or from the project's Settings page in the dashboard.
- Put it in `.env.local`, which should be git-ignored:

  ```bash
  NEXT_PUBLIC_LOGGY_DSN=https://lgy_yourprojectkey@loggy-next.vercel.app
  ```

- Tell the user to add the same variable on their host for deploys, such as the project settings on Vercel.
- The DSN isn't a secret. Browsers send logs with it, and it can only send logs and read flags.

## 3. The component

Add `<Loggy />` once, in the root layout, inside `<body>`:

```tsx
// app/layout.tsx
import { Loggy } from "loggy-next";

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        {children}
        <Loggy />
      </body>
    </html>
  );
}
```

It renders nothing, and it works in a Server Component layout.

## 4. instrumentation.ts (recommended)

Without this file, server logs written before the first page render are missed. That includes static pages and Route Handlers. With it, logging starts when the server boots, it records the route of uncaught server errors, and it writes one log per request: method, status, duration, IP, headers, body, and the error response. From loggy-next 0.7 on, each request log also records where the time went: every `fetch` with its length and cache status, and the render or Route Handler around them.

```ts
// instrumentation.ts, next to the app folder (src/instrumentation.ts when the app uses src/)
export { register, onRequestError } from "loggy-next/instrumentation";
```

If the file already exists:

- Call `register` from loggy-next inside the existing `register`, and do the same for `onRequestError`.
- If the app already registers its own OpenTelemetry, for example `@vercel/otel` or Sentry, Loggy leaves it alone and doesn't write request logs.

## 5. Check it works

1. Run the app and open a page.
2. Call `search_logs` with `range: "15m"` on the project. Browser `console.warn` and `console.error` show up, and so does a request log per request when `instrumentation.ts` is in place.
3. If nothing arrives, check that the DSN variable is set where the app actually runs, and that the server was restarted after `.env.local` changed.

## Writing logs on purpose

Plain `console.*` keeps working:

- On servers, every `console.*` call is sent.
- In browsers, only `console.warn` and `console.error` are sent, unless `browserConsoleLevel` says otherwise.

For logs worth searching, use `log` with structured data. It works in Server Components, Client Components, Route Handlers and Server Actions:

```ts
import { log } from "loggy-next";

log.info("Order created", { orderId, total });
log.error("Payment failed", error);
```

- Levels are `debug`, `info`, `warn` and `error`.
- The fields are searchable, for example `data.orderId:ord_1`.
- In a short-lived script, call `await log.flush()` before it exits.
- Don't log passwords, tokens or personal data. Loggy redacts common secrets, but it can't recognize everything.

## Options

All optional, on `<Loggy />`:

```tsx
<Loggy
  dsn="https://lgy_key@host" // instead of NEXT_PUBLIC_LOGGY_DSN
  environment="staging"      // instead of VERCEL_ENV or NODE_ENV
  minLevel="info"            // ignore anything below this level, everywhere
  browserConsoleLevel="info" // also send console.log and console.info from browsers
  context={{ key: user.id }} // who the browser's flags are for
  release="v1.4.2"           // instead of LOGGY_RELEASE or Vercel's commit
  sessionCookie={false}      // no loggy_sid cookie, so server logs have no session
/>
```

Environment variables:

| Variable | What it does |
|---|---|
| `NEXT_PUBLIC_LOGGY_DSN` | Which project receives the logs. Required. |
| `LOGGY_SERVER_KEY` | Lets the server evaluate flags itself. Secret, server-only, and never prefixed `NEXT_PUBLIC_`. It is created on the project's Settings page; the tools can't make one. |
| `LOGGY_RELEASE`, `NEXT_PUBLIC_LOGGY_RELEASE` | The release name on each log. On Vercel the commit is used without it. |
| `LOGGY_REQUEST_LOGS=false` | Turns off request logs. |
| `LOGGY_REQUEST_BODIES=false` | Leaves request and response bodies out of request logs. |
| `LOGGY_REQUEST_IPS=false` | Leaves IP addresses out of request logs. |

Request logs record IP addresses, headers and bodies, which often hold personal data. If the user's privacy rules don't allow that, use the variables above.

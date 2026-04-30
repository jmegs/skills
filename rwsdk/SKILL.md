---
name: rwsdk
description: Build full-stack web applications with RedwoodSDK (rwsdk) on Cloudflare Workers using TypeScript and React Server Components. Use when working in projects that import from "rwsdk/*" or when the user mentions RedwoodSDK/rwsdk, including routing (defineApp, route, prefix, render, except, linkFor), middleware and interrupters, React Server Components, client components ("use client"), server functions ("use server", serverQuery, serverAction, requestInfo/getRequestInfo), Cloudflare bindings (D1, R2, Queues, Cron, Email Workers), session/auth with defineDurableSession, opt-in passkey addon, security headers/CSP nonce, environment variables, deployment, migration, or create-rwsdk.
---

# RedwoodSDK

React framework for Cloudflare. Starts as a Vite plugin; enables SSR, React Server Components, server functions, and streaming responses. Router and middleware use standard Web APIs: `Request`, `Response`, `URL`, headers, and Cloudflare bindings.

Current docs source: `https://docs.rwsdk.com/llms.txt`. For API details, fetch current docs with ctx7 first.

## Project Shape

```txt
src/
├── worker.tsx              # Entry: defineApp, middleware, routes, Worker export
├── client.tsx              # initClient/initClientNavigation
├── app/
│   ├── Document.tsx        # HTML shell used by render()
│   ├── headers.ts          # app-owned security/header middleware
│   └── pages/
│       └── <section>/
│           ├── routes.ts   # co-located routes
│           └── Page.tsx
├── db/                     # optional query layer; SDK is DB-agnostic
└── sessions/               # optional Durable Object session classes
```

## Quick Reference

### Worker Entry

```tsx
import { defineApp } from "rwsdk/worker";
import { route, render, prefix, except } from "rwsdk/router";
import { Document } from "@/app/Document";
import { setCommonHeaders } from "@/app/headers";
import { routes as blogRoutes } from "@/app/pages/blog/routes";

export const app = defineApp([
  setCommonHeaders(),
  async ({ ctx, request, response, isAction }) => {
    // Global middleware. Server Action requests also pass through here.
    // Use isAction to skip page-only work.
  },
  except((error) => {
    console.error(error);
    return new Response("Internal Server Error", { status: 500 });
  }),
  render(Document, [
    route("/", () => <HomePage />),
    prefix("/blog", blogRoutes),
  ]),
]);

export default {
  fetch: app.fetch,
} satisfies ExportedHandler<Env>;
```

Simple apps can also `export default defineApp([...])`. If using Cron, Queues, Email, custom global error handling, or exported `app` for `linkFor`, export an object with `fetch: app.fetch`.

### Routes

```tsx
route("/users/:id", ({ params }) => <UserProfile id={params.id} />);
route("/files/*", ({ params }) => new Response(params.$0));
route("/api/data", () => Response.json({ ok: true }));
route("/api/users", {
  get: listUsers,
  head: listUsers, // HEAD is explicit; not auto-mapped from GET.
  post: [requireAuth, createUser],
  config: { disableOptions: true, disable405: true },
});
route("/admin", [requireAuth, isAdmin, AdminPage]); // interrupters run L->R
```

Routes match in definition order. Use standard `new URL(request.url).searchParams` for query strings.

### Server Functions

```tsx
// actions.ts
"use server";

import { getRequestInfo, serverAction, serverQuery } from "rwsdk/worker";

export const getItems = serverQuery(async () => {
  const { ctx } = getRequestInfo();
  return listItemsFor(ctx.user.id);
});

export const createItem = serverAction(async (formData: FormData) => {
  const { ctx } = getRequestInfo();
  await createItemFor(ctx.user.id, String(formData.get("name")));
});
```

Use `serverQuery` for data-only GET calls without page rehydration. Use `serverAction` for mutations; default POST triggers rehydration.

### Client Entry

```tsx
import { initClient, initClientNavigation } from "rwsdk/client";

initClient({
  onActionResponse: (response) => {
    // Return true to prevent default redirect handling.
  },
});

initClientNavigation({
  scrollToTop: true,
  scrollBehavior: "instant",
});
```

## Data & Bindings

RedwoodSDK is database-agnostic. Use Web/Cloudflare APIs directly and choose the query layer per project:

```tsx
import { env } from "cloudflare:workers";

const { results } = await env.DB.prepare("SELECT * FROM todos").all();
```

D1 with native SQL or Drizzle is common on Cloudflare, but not required. Experimental SDK database and realtime primitives are opt-in only; do not make them the default path.

## References

- **Routing**: [references/routing.md](references/routing.md) — matching, methods, interrupters, `except`, `render`, `linkFor`, prefetch
- **Middleware**: [references/middleware.md](references/middleware.md) — global middleware, context, action pipeline, headers
- **Interrupters**: [references/interruptors.md](references/interruptors.md) — route-scoped auth, validation, RBAC; "interruptors" accepted user spelling
- **React/RSC**: [references/react.md](references/react.md) — server/client components, server functions, `serverQuery`/`serverAction`, responses
- **Client**: [references/client.md](references/client.md) — `initClient`, `initClientNavigation`, `navigate`
- **Database**: [references/database.md](references/database.md) — DB-agnostic approach, D1 native, Drizzle, opt-in experimental `rwsdk/db`
- **Authentication**: [references/auth.md](references/auth.md) — `defineDurableSession`, session DO, opt-in passkey addon
- **Platform**: [references/platform.md](references/platform.md) — R2, Queues, Email Workers, Cron
- **Deployment**: [references/deployment.md](references/deployment.md) — env vars, hosting, security headers, CSP/nonce
- **Migration**: [references/migration.md](references/migration.md) — 0.x to 1.x, `create-rwsdk`
- **Experimental Realtime**: [references/realtime.md](references/realtime.md) — opt-in `useSyncedState`; not happy path

## Guidelines

1. Fetch current docs for API questions; prefer `https://docs.rwsdk.com/llms.txt` and ctx7 `/websites/rwsdk`.
2. Use Web APIs over external dependencies unless the project already chose a dependency.
3. Keep route definitions explicit; co-locate section route arrays and compose with `prefix`.
4. Use middleware for global concerns; use interrupters for route-specific concerns.
5. Mutate `ctx` in middleware/interrupters; short-circuit by returning a `Response`.
6. Default to server components; use `"use client"` only for browser interactivity.
7. Access request context in server functions via `getRequestInfo()` or `requestInfo`.
8. Access Cloudflare bindings via `import { env } from "cloudflare:workers"`.
9. Keep database guidance agnostic; do not prescribe ORM or experimental storage by default.
10. Mark experimental APIs explicitly and keep them out of the happy path.

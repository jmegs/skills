---
name: rwsdk
description: Build full-stack web applications with RedwoodSDK (rwsdk) on Cloudflare Workers using TypeScript and React Server Components. Use when working with rwsdk projects, including routing (defineApp, route, prefix, render), middleware (worker.tsx, sessions, auth), interruptors (auth guards, validation, RBAC), React Server Components, client components ("use client"), server functions ("use server", serverQuery, serverAction, requestInfo), database (D1 — query with native bindings, Drizzle, or your preferred query layer), authentication (defineDurableSession, passkey/WebAuthn), storage (R2), background jobs (Queues), email (Email Workers), cron triggers, security (CSP, nonce), environment variables, or deployment. Trigger when the user mentions RedwoodSDK, rwsdk, or is working in a project that imports from "rwsdk/*".
---

# RedwoodSDK

React framework for Cloudflare. Server-side rendering, RSC, server functions, streaming. Zero magic — no hidden codegen or transpilation side effects. Composability over configuration. Web-first (native Request/Response/URL).

## Project Structure

```
src/
├── worker.tsx              # Entry: defineApp, middleware, routes
├── client.tsx              # Client entry
├── db/
│   ├── schema.ts           # Database schema (e.g., Drizzle table defs)
│   └── index.ts            # DB client instance
├── sessions/
│   └── UserSession.ts      # Session Durable Object
└── app/
    ├── Document.tsx         # HTML shell (used with render())
    ├── interruptors.ts      # Shared interruptors (auth, validation)
    └── pages/
        └── <section>/
            ├── routes.ts    # Co-located routes
            └── Page.tsx
```

## Quick Reference

### Worker Entry (`worker.tsx`)

```tsx
import { defineApp, route, render, prefix } from "rwsdk/router";
import { setCommonHeaders } from "rwsdk/headers";
import { Document } from "@/app/Document";

const app = defineApp([
  setCommonHeaders(),
  async ({ ctx, request, response }) => {
    // Session/auth middleware — see references/auth.md
  },
  render(Document, [
    route("/", HomePage),
    prefix("/blog", blogRoutes),
  ]),
]);

// Simple app:
export default app;

// With queues/cron/email, export object instead:
export default {
  fetch: app.fetch,
  async queue(batch) { /* ... */ },
  async scheduled(controller) { /* ... */ },
} satisfies ExportedHandler<Env>;
```

### Route Handlers

Receive `{ request, params, ctx }`. Return JSX, `Response`, or `Response.json()`.

```tsx
route("/users/:id", ({ params }) => <UserProfile id={params.id} />)
route("/api/data", () => Response.json({ ok: true }))
route("/api/users", { get: listUsers, post: createUser, delete: deleteUser })
route("/admin", [requireAuth, isAdmin, AdminPage])  // interruptors run L→R
```

### Server Functions

```tsx
// actions.ts
"use server";
import { serverAction, serverQuery, requestInfo } from "rwsdk/worker";

// Mutations (POST, triggers rehydration):
export const createItem = serverAction(async (formData: FormData) => {
  const { ctx } = requestInfo;
  // ...
});

// Reads (GET, no rehydration):
export const getItems = serverQuery(async () => {
  return db.select().from(items);
});
```

### Database

D1 (Cloudflare's native SQLite) is the default database. rwsdk doesn't prescribe how you query it — native prepared statements, Drizzle, whatever you prefer.

```tsx
// Native D1 binding:
import { env } from "cloudflare:workers";
const { results } = await env.DB.prepare("SELECT * FROM todos").all();

// Or with Drizzle ORM:
import { drizzle } from "drizzle-orm/d1";
const db = drizzle(env.DB, { schema });
const allTodos = await db.select().from(todos);
```

See [references/database.md](references/database.md) for full setup (D1 creation, migrations, query layer options).

### Environment Variables

```tsx
import { env } from "cloudflare:workers";
// Access bindings directly: env.DB, env.R2, env.QUEUE, env.EMAIL
```

### Context Types (`global.d.ts`)

```tsx
import { DefaultAppContext } from "rwsdk/worker";
interface AppContext { user?: User; session?: { userId: string | null }; }
declare module "rwsdk/worker" { interface DefaultAppContext extends AppContext {} }
```

## References

- **Routing**: See [references/routing.md](references/routing.md) — path matching, HTTP methods, query params, `linkFor`, prefetching, co-located routes
- **Middleware**: See [references/middleware.md](references/middleware.md) — global middleware, session setup, context population
- **Interruptors**: See [references/interruptors.md](references/interruptors.md) — auth guards, Zod validation, RBAC, composition
- **React/RSC**: See [references/react.md](references/react.md) — server vs client components, `serverQuery`/`serverAction`, `renderToStream`/`renderToString`, context
- **Database**: See [references/database.md](references/database.md) — D1 setup, query layer options (native, Drizzle, experimental Kysely)
- **Authentication**: See [references/auth.md](references/auth.md) — `defineDurableSession`, session DO, passkey addon
- **Platform**: See [references/platform.md](references/platform.md) — R2 storage, Queues, Email Workers, Cron Triggers
- **Deployment**: See [references/deployment.md](references/deployment.md) — env vars, hosting, staging, security headers, CSP/nonce

## Guidelines

1. Use Web APIs over external deps (`fetch` not Axios, `Response.json()` for JSON).
2. Co-locate routes in `src/app/pages/<section>/routes.ts`, import with `prefix`.
3. Default to server components; `"use client"` only for interactivity.
4. Keep client components small to minimize JS bundle.
5. Interruptors for per-route concerns; middleware for global concerns.
6. Access context via `requestInfo` in server functions, via props in server components.
7. Use `import { env } from "cloudflare:workers"` for bindings — no env threading needed.
8. D1 is the default database. rwsdk doesn't care how you query it — native prepared statements, Drizzle, etc.
9. Use `serverAction` for mutations (POST), `serverQuery` for reads (GET).

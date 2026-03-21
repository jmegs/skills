# Middleware

Middleware functions run on every request before route handlers. Use for security headers, CORS, caching, logging, sessions, and authentication.

## Middleware vs Interruptors

- **Middleware**: Every request. Defined in `defineApp` before routes. Cross-cutting concerns.
- **Interruptors**: Per-route. Defined in route arrays. Route-specific concerns.

## Basic Structure

```tsx
import { defineApp, ErrorResponse } from "rwsdk/worker";
import { route, render } from "rwsdk/router";
import { setCommonHeaders } from "rwsdk/headers";
import { defineDurableSession } from "rwsdk/auth";
import { env } from "cloudflare:workers";
import { Document } from "@/app/Document";

export const sessionStore = defineDurableSession({
  sessionDurableObject: env.USER_SESSION_DO,
});

const app = defineApp([
  // 1. Security headers
  setCommonHeaders(),

  // 2. Session + auth middleware
  async ({ ctx, request, response }) => {
    try {
      ctx.session = await sessionStore.load(request);
    } catch (error) {
      if (error instanceof ErrorResponse && error.code === 401) {
        await sessionStore.remove(request, response.headers);
        response.headers.set("Location", "/user/login");
        return new Response(null, { status: 302, headers: response.headers });
      }
      throw error;
    }

    if (ctx.session?.userId) {
      const [user] = await db
        .select()
        .from(users)
        .where(eq(users.id, ctx.session.userId));
      ctx.user = user;
    }
  },

  // 3. Routes wrapped in Document
  render(Document, [
    route("/", HomePage),
    // ...
  ]),
]);
```

## Context

The `ctx` object is mutable and request-scoped. Populate it in middleware, access it in route handlers, interruptors, and server components (via props). Server actions access it via `requestInfo.ctx`.

Server actions also pass through the middleware pipeline.

## Extending Context Types

Define in `global.d.ts`:

```tsx
import { DefaultAppContext } from "rwsdk/worker";

interface AppContext {
  user?: User;
  session?: { userId: string | null };
}

declare module "rwsdk/worker" {
  interface DefaultAppContext extends AppContext {}
}
```

## Key Points

1. `setCommonHeaders()` from `rwsdk/headers` adds security headers automatically.
2. Middleware has access to `ctx`, `request`, and `response`.
3. Return a `Response` to short-circuit (e.g., redirect on auth failure).
4. Populate `ctx` with session/user data for downstream routes.
5. Worker entry is `src/worker.tsx`.

# Middleware

Middleware runs before route matching. Use it for cross-cutting concerns: headers, sessions, auth context, CORS, logging, cache policy, and request metadata.

## Middleware vs Interrupters

- Middleware: global; declared in `defineApp` before routes.
- Interrupters: route-scoped; declared in route arrays.

Server Action requests also pass through the global middleware pipeline. Use `isAction` for page-only behavior.

## Basic Structure

```tsx
import { defineApp, ErrorResponse } from "rwsdk/worker";
import { route, render } from "rwsdk/router";
import { defineDurableSession } from "rwsdk/auth";
import { env } from "cloudflare:workers";
import { setCommonHeaders } from "@/app/headers";
import { Document } from "@/app/Document";

export const sessionStore = defineDurableSession({
  sessionDurableObject: env.USER_SESSION_DO,
});

export const app = defineApp([
  setCommonHeaders(),
  async ({ ctx, request, response, isAction }) => {
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

    if (!isAction) {
      response.headers.set("X-Frame-Options", "DENY");
    }
  },
  render(Document, [route("/", HomePage)]),
]);
```

## Context

`ctx` is mutable and request-scoped. Populate it in middleware; access it in route handlers, interrupters, server components, and server functions.

```tsx
async ({ ctx }) => {
  ctx.user = await findUser(ctx.session?.userId);
}
```

Extend app context types in `global.d.ts`.

```ts
import { DefaultAppContext } from "rwsdk/worker";

interface AppContext {
  user?: User;
  session?: { userId: string | null };
}

declare module "rwsdk/worker" {
  interface DefaultAppContext extends AppContext {}
}
```

## Headers

RedwoodSDK exposes `response.headers`; mutate it directly.

```tsx
import type { RouteMiddleware } from "rwsdk/worker";

export const setCommonHeaders =
  (): RouteMiddleware =>
  ({ response, rw: { nonce } }) => {
    response.headers.set("X-Frame-Options", "DENY");
    response.headers.set("X-Content-Type-Options", "nosniff");
    response.headers.set(
      "Content-Security-Policy",
      `default-src 'self'; script-src 'self' 'nonce-${nonce}'; object-src 'none';`,
    );
  };
```

Do not use removed context `headers`; set outgoing headers via `response.headers`.

# Routing & Request Handling

Import router helpers from `rwsdk/router`. Import `defineApp` and worker-side helpers from `rwsdk/worker`.

## Matching

Routes match in definition order. Trailing slashes are normalized.

```tsx
import { defineApp } from "rwsdk/worker";
import { route, prefix, render } from "rwsdk/router";

export default defineApp([
  route("/", () => <Home />),
  route("/about", () => new Response("About")),
  route("/users/:id", ({ params }) => <User id={params.id} />),
  route("/files/*", ({ params }) => new Response(params.$0)),
]);
```

Patterns:

- Static: `/about`
- Parameter: `/users/:id`
- Wildcard: `/files/*`, exposed as `params.$0`, `params.$1`, etc.

## Query Parameters

Use the standard URL API.

```tsx
route("/search", ({ request }) => {
  const url = new URL(request.url);
  const q = url.searchParams.get("q") ?? "";
  const tags = url.searchParams.getAll("tag");
  return Response.json({ q, tags });
});
```

## HTTP Method Routing

Pass method handlers to one route. Method handlers can be functions or arrays of interrupters plus final handler.

```tsx
route("/api/users", {
  get: listUsers,
  head: listUsers,
  post: [requireAuth, validateUser, createUser],
  put: updateUser,
  patch: patchUser,
  delete: deleteUser,
  custom: {
    report: reportUsers,
  },
  config: {
    disableOptions: true,
    disable405: true,
  },
});
```

Defaults:

- `OPTIONS` returns `204 No Content` with `Allow`.
- Unsupported methods return `405 Method Not Allowed`.
- `HEAD` is not mapped to `GET`; add `head` explicitly.
- `custom` handles non-standard methods case-insensitively.

## Interrupters

Interrupters are route-scoped middleware. They run left to right before the final handler. Mutate `ctx` to share data; return a `Response` to stop the chain.

```tsx
async function requireAuth({ ctx }) {
  if (!ctx.user) return new Response("Unauthorized", { status: 401 });
}

route("/admin", [requireAuth, ({ ctx }) => <Admin user={ctx.user} />]);
```

The docs spell this "interrupters"; older/user wording may say "interruptors".

## Middleware & Context

Middleware is declared in `defineApp` before routes. It runs before route matching and also runs for Server Action requests. Use `isAction` to skip page-only work.

```tsx
export default defineApp([
  async ({ ctx, request, response, isAction }) => {
    ctx.session = await loadSession(request);
    if (!isAction) response.headers.set("X-Page-Request", "1");
  },
  route("/hello", ({ ctx }) => new Response(`Hello ${ctx.user?.name ?? "anon"}`)),
]);
```

Extend context types in `global.d.ts`:

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

## Documents & render

Documents define the HTML shell. Include the client entry if you need hydration.

```tsx
export function Document({ children, rw }) {
  return (
    <html lang="en">
      <head>
        <meta charSet="utf-8" />
        <meta name="viewport" content="width=device-width, initial-scale=1" />
        <script type="module" src="/src/client.tsx" nonce={rw.nonce} />
      </head>
      <body>{children}</body>
    </html>
  );
}

render(Document, [route("/", Home)]);
```

`render(Document, routes, options)` supports:

- `rscPayload` default `true`; disable only for static output with no client interactivity.
- `ssr` default `true`; set `false` for client-only routes that need browser APIs.

## Error Handling

Use `except` for route-tree error handling. It catches errors from middleware, route handlers, route arrays, and RSC actions. Nested handlers bubble by returning `void`.

```tsx
import { except, route } from "rwsdk/router";

export default defineApp([
  except((error) => {
    console.error(error);
    return <ErrorPage error={error} />;
  }),
  route("/", Home),
]);
```

For structured status errors, throw or return `ErrorResponse` from `rwsdk/worker` where appropriate.

## Request Info

Use `requestInfo` or `getRequestInfo()` from `rwsdk/worker` in server functions.

```tsx
import { getRequestInfo } from "rwsdk/worker";

export async function myServerFunction() {
  const { request, response, ctx, rw, cf } = getRequestInfo();
  response.headers.set("Cache-Control", "no-store");
}
```

`getRequestInfo()` throws outside a request lifecycle. `requestInfo` exposes request-scoped fields.

## linkFor

Create a typed link helper from the exported app type.

```tsx
// worker.tsx
export const app = defineApp([render(Document, [route("/users/:id", UserPage)])]);
export default { fetch: app.fetch } satisfies ExportedHandler<Env>;

// app/shared/links.ts
import { linkFor } from "rwsdk/router";
import type * as Worker from "../../worker";

type App = typeof Worker.app;
export const link = linkFor<App>();

link("/users/:id", { id: user.id });
```

If the worker default export is the app itself, use `typeof Worker.default`.

## Prefetch

When client navigation is enabled, add `<link rel="x-prefetch" href={href} />` for likely next pages. RedwoodSDK fetches the RSC payload in the background with `__rsc` and `x-prefetch: true`, then uses the browser Cache API for the next navigation.

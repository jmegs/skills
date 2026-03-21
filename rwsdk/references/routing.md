# Routing & Request Handling

## Route Definition

Use `route` from `rwsdk/router`. Routes match in definition order. Trailing slashes normalized.

### Static Paths

```tsx
route("/", () => <>Home</>)
route("/about", () => <>About</>)
```

### Dynamic Parameters

```tsx
route("/users/:id", ({ params }) => <>User {params.id}</>)
route("/posts/:postId/comments/:commentId", ({ params }) => (
  <>Comment {params.commentId} on Post {params.postId}</>
))
```

### Wildcards

```tsx
route("/files/*", ({ params }) => <>File: {params.$0}</>)
route("/docs/*/version/*", ({ params }) => <>Doc: {params.$0}, Version: {params.$1}</>)
```

## HTTP Method Routing

Pass an object with method keys to handle different verbs on the same path. Supports `get`, `post`, `delete`, `put`, `patch`, `custom`. Automatic OPTIONS and 405 handling.

```tsx
route("/api/users", {
  get: () => Response.json(users),
  post: async ({ request }) => {
    const data = await request.json();
    return new Response("Created", { status: 201 });
  },
  delete: () => new Response(null, { status: 204 }),
})

// Method handlers can include interruptors:
route("/api/admin", {
  get: [requireAuth, listAdmins],
  post: [requireAuth, isAdmin, createAdmin],
})
```

## Response Types

```tsx
// JSX
route("/page", () => <MyComponent />)

// JSON
route("/api/data", ({ params }) => Response.json({ id: params.id }))

// Plain text
route("/health", () => new Response("OK", {
  headers: { "Content-Type": "text/plain" },
}))

// With Document wrapper
render(Document, [
  route("/", () => <>Home</>),
  route("/about", () => <>About</>),
])
```

## Query Parameters

Standard Web Request URL API:

```tsx
route("/api/search", ({ request }) => {
  const url = new URL(request.url);
  const query = url.searchParams.get("q") || "";
  const page = parseInt(url.searchParams.get("page") || "1");
  const tags = url.searchParams.getAll("tag");
  return Response.json({ query, page, tags });
})
```

## Co-located Routes with Prefix

`src/app/pages/blog/routes.ts`:

```tsx
import { route } from "rwsdk/router";
import { requireAuth } from "@/app/interruptors";
import { BlogList } from "./BlogList";
import { BlogPost } from "./BlogPost";

export const routes = [
  route("/", BlogList),
  route("/post/:postId", BlogPost),
  route("/post/:postId/edit", [requireAuth, BlogEditPage]),
];
```

Import in `worker.tsx`:

```tsx
import { routes as blogRoutes } from "@/app/pages/blog/routes";

// Inside defineApp:
render(Document, [
  route("/", HomePage),
  prefix("/blog", blogRoutes),
])
```

## Type-Safe Links with `linkFor`

```tsx
import { linkFor } from "rwsdk/router";
import type * as Worker from "../../worker";
type App = typeof Worker.default;

export const link = linkFor<App>();

// Usage — type-checked against your actual routes:
const home = link("/");
const userProfile = link("/users/:id", { id: user.id });
const callDetails = link("/calls/details/:id", { id: call.id });
```

## Prefetching

Include `<link rel="x-prefetch">` elements. RedwoodSDK scans for these and issues background GET requests with `__rsc` query param and `x-prefetch: true` header, caching responses for faster navigation.

```tsx
<link rel="x-prefetch" href={link("/about")} />
```

## Documents

The HTML shell for your app. Used with `render()`:

```tsx
export const Document: React.FC<{ children: React.ReactNode; rw: { nonce: string } }> = ({
  children,
  rw,
}) => (
  <html lang="en">
    <head>
      <meta charSet="utf-8" />
      <meta name="viewport" content="width=device-width, initial-scale=1" />
      <script type="module" src="/src/client.tsx"></script>
    </head>
    <body>{children}</body>
  </html>
);
```

The `rw` prop provides `nonce` for inline script CSP compliance.

## Request Info

`requestInfo` and `getRequestInfo()` from `rwsdk/worker` provide access to `request`, `response`, `ctx`, `rw`, `cf`. Mutate `response.status` and `response.headers` to configure outgoing response.

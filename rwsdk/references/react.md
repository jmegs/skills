# React Server Components & Server Functions

## Server Components

Components are server components by default. They render on the server, stream HTML, can be async, and cannot use browser-only APIs, state, effects, or event handlers.

```tsx
export async function TodoList({ ctx }) {
  const todos = await getTodosForUser(ctx.user.id);

  return (
    <ol>
      {todos.map((todo) => (
        <li key={todo.id}>{todo.title}</li>
      ))}
    </ol>
  );
}
```

Wrap async server components in `Suspense` when you want a loading boundary.

```tsx
export function TodoPage({ ctx }) {
  return (
    <Suspense fallback={<div>Loading...</div>}>
      <TodoList ctx={ctx} />
    </Suspense>
  );
}
```

## Client Components

Use `"use client"` for browser interactivity: state, effects, event handlers, DOM APIs, and client-side hooks.

```tsx
"use client";

import { useState } from "react";

export function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount((n) => n + 1)}>Count: {count}</button>;
}
```

Keep client components small; they add to the browser bundle.

## Server Functions

Server functions live in `"use server"` files and can be called from client components.

```tsx
"use server";

import { getRequestInfo } from "rwsdk/worker";

export async function addTodo(formData: FormData) {
  const { ctx } = getRequestInfo();
  const title = String(formData.get("title") ?? "");
  await createTodo({ title, userId: ctx.user.id });
}
```

```tsx
"use client";

import { addTodo } from "./actions";

export function AddTodo() {
  return (
    <form action={addTodo}>
      <input name="title" />
      <button type="submit">Add</button>
    </form>
  );
}
```

## serverQuery

Use `serverQuery` for data-only reads. Default method is GET. It avoids page rehydration by returning only the action result.

```tsx
"use server";

import { serverQuery } from "rwsdk/worker";

export const getTodos = serverQuery(async (userId: string) => {
  return listTodos(userId);
});

export const getSecretData = serverQuery([
  requireAuth,
  async () => {
    return "Secret Data";
  },
]);
```

`serverQuery` sends an `x-rsc-data-only: true` request and RedwoodSDK skips expensive page rendering.

## serverAction

Use `serverAction` for mutations. Default method is POST. It rehydrates and re-renders with updated server state.

```tsx
"use server";

import { serverAction } from "rwsdk/worker";

export const createTodo = serverAction(async (title: string) => {
  await insertTodo({ title });
});

export const deleteTodo = serverAction([
  requireAuth,
  async (id: string) => {
    await deleteTodoById(id);
  },
]);

export const searchTodos = serverAction(
  async (query: string) => {
    return search(query);
  },
  { method: "GET" },
);
```

## Request Context

Server components receive `ctx` from route props. Server functions use `requestInfo` or `getRequestInfo()` from `rwsdk/worker`.

```tsx
import { getRequestInfo, requestInfo } from "rwsdk/worker";

const { ctx } = getRequestInfo();
const request = requestInfo.request;
```

Prefer `getRequestInfo()` when you want a hard failure outside request scope.

## Returning Responses

Server functions can return `Response` objects for redirects or custom metadata.

```tsx
"use server";

export async function createPost(formData: FormData) {
  const id = await savePost(formData);
  return Response.redirect(`/posts/${id}`, 303);
}
```

RedwoodSDK handles 3xx `Location` redirects on the client. Other responses expose status and headers to client response handling.

## Intercepting Action Responses

Configure this in the client entry.

```tsx
import { initClient } from "rwsdk/client";

initClient({
  onActionResponse: (response) => {
    if (response.status === 409) {
      showConflictToast();
      return true;
    }
  },
});
```

Return `true` from `onActionResponse` to prevent default redirect handling.

## Manual Rendering

For custom worker responses, use worker rendering helpers when needed.

```tsx
import { renderToStream, renderToString } from "rwsdk/worker";

const stream = await renderToStream(<NotFound />, { Document });
return new Response(stream, { status: 404 });

const html = await renderToString(<EmailPreview />, { Document });
return new Response(html, { headers: { "Content-Type": "text/html" } });
```

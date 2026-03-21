# React Server Components & Server Functions

## Server Components (Default)

All components are server components by default. They render on the server, stream as HTML. Can be async. Cannot use state, effects, or event handlers.

```tsx
import { db } from "@/db";
import { todos } from "@/db/schema";
import { eq } from "drizzle-orm";

export async function TodoList({ ctx }) {
  const items = await db.select().from(todos).where(eq(todos.userId, ctx.user.id));

  return (
    <ol>
      {items.map((todo) => <li key={todo.id}>{todo.title}</li>)}
    </ol>
  );
}
```

Wrap async server components in Suspense:

```tsx
export function TodoPage({ ctx }) {
  return (
    <div>
      <h1>Todos</h1>
      <Suspense fallback={<div>Loading...</div>}>
        <TodoList ctx={ctx} />
      </Suspense>
    </div>
  );
}
```

## Client Components

Mark with `"use client"`. Required for interactivity, browser APIs, event handlers, state, effects.

```tsx
"use client";

import { useState } from "react";

export function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(c => c + 1)}>Count: {count}</button>;
}
```

Keep client components small — they add to the JS bundle.

## Server Functions

### Basic Pattern

```tsx
// src/app/actions.ts
"use server";

import { requestInfo } from "rwsdk/worker";

export async function addTodo(formData: FormData) {
  const { ctx } = requestInfo;
  const title = formData.get("title") as string;
  await db.insert(todos).values({
    id: crypto.randomUUID(),
    title,
    userId: ctx.user.id,
  });
}
```

Client component calling server function:

```tsx
"use client";

import { addTodo } from "../actions";

export function AddTodoForm() {
  return (
    <form action={addTodo}>
      <input type="text" name="title" />
      <button type="submit">Add</button>
    </form>
  );
}
```

### `serverQuery` and `serverAction`

Use these wrappers for better control over HTTP method and rehydration behavior.

**`serverQuery`** — GET requests, no page rehydration:

```tsx
"use server";

import { serverQuery } from "rwsdk/worker";

export const getTodos = serverQuery(async (userId: string) => {
  return db.select().from(todos).where(eq(todos.userId, userId));
});

// With interruptors:
export const getSecretData = serverQuery([
  async () => {
    if (!isAuthenticated()) throw new Response("Unauthorized", { status: 401 });
  },
  async () => "Secret Data",
]);
```

**`serverAction`** — POST requests, triggers page rehydration:

```tsx
"use server";

import { serverAction } from "rwsdk/worker";

export const createTodo = serverAction(async (title: string) => {
  await db.insert(todos).values({
    id: crypto.randomUUID(),
    title,
    completed: 0,
  });
});

// With interruptors:
export const deleteTodo = serverAction([
  requireAuth,
  async (id: string) => {
    await db.delete(todos).where(eq(todos.id, id));
  },
]);

// Force GET method on an action:
export const searchTodos = serverAction(
  async (query: string) => { /* ... */ },
  { method: "GET" },
);
```

### Returning Responses

Server functions can return Response objects for redirects or custom behavior:

```tsx
"use server";

export async function createPost(formData: FormData) {
  // ... create post ...
  return Response.redirect("/posts", 303);
}
```

### Intercepting Action Responses (Client)

```tsx
import { initClient } from "rwsdk/client";

initClient({
  onActionResponse: (response) => {
    console.log("Action returned status:", response.status);
    // Return true to prevent default redirect behavior
  },
});
```

## Context Access

- **Server components**: `ctx` passed as props from route handlers
- **Server functions**: `import { requestInfo } from "rwsdk/worker"; const { ctx } = requestInfo;`
- Context is populated by middleware/interruptors, request-scoped

## Manual Rendering

For custom responses (error pages, emails):

```tsx
import { renderToStream, renderToString } from "rwsdk/worker";
import { Document } from "@/app/Document";

// Streaming:
const stream = await renderToStream(<NotFound />, { Document });
return new Response(stream, { status: 404 });

// String:
const html = await renderToString(<NotFound />, { Document });
return new Response(html, { status: 404 });
```

Options: `Document` (wrapper component), `rscPayload` (include RSC payload, default true for stream).

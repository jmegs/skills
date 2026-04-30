# Database

RedwoodSDK is database-agnostic. It runs on Cloudflare Workers, so D1 is a common choice, but the SDK does not prescribe ORM, schema tool, or query layer.

Use the project’s existing data layer when present. For new code, choose the smallest query layer that fits: native D1 prepared statements, Drizzle, a remote API, external database, or another Cloudflare binding.

## D1 Binding

Create a D1 database:

```bash
npx wrangler d1 create my-app-db
```

Add binding to `wrangler.jsonc`:

```jsonc
{
  "d1_databases": [
    {
      "binding": "DB",
      "database_name": "my-app-db",
      "database_id": "<id-from-command>"
    }
  ]
}
```

Regenerate types after binding changes:

```bash
pnpm generate
# or
npx wrangler types
```

Access bindings via Cloudflare Workers env:

```ts
import { env } from "cloudflare:workers";

const db = env.DB;
```

## Native D1

Native D1 is zero extra dependency and works well for direct SQL.

```ts
import { env } from "cloudflare:workers";

await env.DB.prepare("INSERT INTO todos (id, title, completed) VALUES (?, ?, ?)")
  .bind(crypto.randomUUID(), title, 0)
  .run();

const todo = await env.DB.prepare("SELECT * FROM todos WHERE id = ?")
  .bind(todoId)
  .first();

const { results } = await env.DB.prepare("SELECT * FROM todos ORDER BY created_at DESC").all();
```

Manage schema with D1 migrations:

```bash
npx wrangler d1 migrations create my-app-db init
npx wrangler d1 migrations apply my-app-db --local
npx wrangler d1 migrations apply my-app-db --remote
```

## Drizzle

Use Drizzle when the project wants schema-as-code and typed query builder ergonomics.

```bash
pnpm add drizzle-orm
pnpm add -D drizzle-kit
```

```ts
// src/db/schema.ts
import { integer, sqliteTable, text } from "drizzle-orm/sqlite-core";

export const todos = sqliteTable("todos", {
  id: text("id").primaryKey(),
  title: text("title").notNull(),
  completed: integer("completed").notNull().default(0),
  createdAt: text("created_at").notNull(),
});
```

```ts
// src/db/index.ts
import { env } from "cloudflare:workers";
import { drizzle } from "drizzle-orm/d1";
import * as schema from "./schema";

export const db = drizzle(env.DB, { schema });
```

```ts
import { eq } from "drizzle-orm";
import { db } from "@/db";
import { todos } from "@/db/schema";

await db.insert(todos).values({
  id: crypto.randomUUID(),
  title,
  completed: 0,
  createdAt: new Date().toISOString(),
});

const rows = await db.select().from(todos);
const [todo] = await db.select().from(todos).where(eq(todos.id, todoId));
```

## Server Functions

Use the chosen query layer from server components, route handlers, and server functions.

```ts
"use server";

import { getRequestInfo, serverAction, serverQuery } from "rwsdk/worker";

export const getTodos = serverQuery(async () => {
  const { ctx } = getRequestInfo();
  return listTodos(ctx.user.id);
});

export const createTodo = serverAction(async (formData: FormData) => {
  const { ctx } = getRequestInfo();
  await insertTodo({
    title: String(formData.get("title") ?? ""),
    userId: ctx.user.id,
  });
});
```

## Opt-In Experimental: rwsdk/db

`rwsdk/db` is experimental. Do not use it as the default recommendation. Mention it only when the user explicitly asks about the experimental SDK database or the project already uses `rwsdk/db`.

Current docs describe SQLite Durable Objects plus Kysely, with types inferred from migrations. Each database instance is isolated in its own Durable Object. Migration rollback is best effort; write idempotent `down()` functions because SQLite DDL is not fully transactional.

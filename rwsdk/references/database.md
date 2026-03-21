# Database

rwsdk runs on Cloudflare Workers, so **D1** (Cloudflare's native SQLite database) is the natural default. rwsdk doesn't prescribe how you query D1 — use the native binding with prepared statements, Drizzle ORM, or whatever you prefer.

## D1 Setup

### 1. Create Database

```bash
npx wrangler d1 create my-app-db
```

Add the binding to `wrangler.jsonc`:

```jsonc
{
  "d1_databases": [
    { "binding": "DB", "database_name": "my-app-db", "database_id": "<id-from-above>" }
  ]
}
```

Run `pnpm generate` (or `npx wrangler types`) to regenerate the `Env` type with the new binding.

### 2. Access the Binding

```ts
import { env } from "cloudflare:workers";

// env.DB is your D1Database binding
```

## Query Layer Options

### Option A: Native D1 Prepared Statements

Zero dependencies. Just SQL. Good for simple apps or if you want full control.

```ts
import { env } from "cloudflare:workers";

// Create
await env.DB.prepare("INSERT INTO todos (id, text, completed) VALUES (?, ?, ?)")
  .bind(crypto.randomUUID(), "Ship it", 0)
  .run();

// Read
const todo = await env.DB.prepare("SELECT * FROM todos WHERE id = ?")
  .bind(todoId)
  .first();

const allTodos = await env.DB.prepare("SELECT * FROM todos ORDER BY created_at DESC")
  .all();

// Update
await env.DB.prepare("UPDATE todos SET completed = 1 WHERE id = ?")
  .bind(todoId)
  .run();

// Delete
await env.DB.prepare("DELETE FROM todos WHERE id = ?")
  .bind(todoId)
  .run();
```

Manage schema with [D1 migrations](https://developers.cloudflare.com/d1/reference/migrations/):

```bash
npx wrangler d1 migrations create my-app-db init
# edit the generated SQL file, then:
npx wrangler d1 migrations apply my-app-db --local    # dev
npx wrangler d1 migrations apply my-app-db --remote   # production
```

### Option B: Drizzle ORM

Type-safe query builder with schema-as-code. The most popular ORM in the Cloudflare ecosystem.

**Install:**

```bash
pnpm add drizzle-orm
pnpm add -D drizzle-kit
```

**Define schema** (`src/db/schema.ts`):

```ts
import { sqliteTable, text, integer } from "drizzle-orm/sqlite-core";

export const todos = sqliteTable("todos", {
  id: text("id").primaryKey(),
  text: text("text").notNull(),
  completed: integer("completed").notNull().default(0),
  createdAt: text("created_at").notNull(),
});
```

**Configure Drizzle** (`drizzle.config.ts`):

```ts
import { defineConfig } from "drizzle-kit";

export default defineConfig({
  out: "./drizzle",
  schema: "./src/db/schema.ts",
  dialect: "sqlite",
});
```

**Migrations:**

```bash
npx drizzle-kit generate                              # generate SQL from schema
npx wrangler d1 migrations apply my-app-db --local    # dev
npx wrangler d1 migrations apply my-app-db --remote   # production
```

**Create client** (`src/db/index.ts`):

```ts
import { drizzle } from "drizzle-orm/d1";
import { env } from "cloudflare:workers";
import * as schema from "./schema";

export const db = drizzle(env.DB, { schema });
```

**Query:**

```ts
import { db } from "@/db";
import { todos } from "@/db/schema";
import { eq } from "drizzle-orm";

await db.insert(todos).values({ id: crypto.randomUUID(), text: "Ship it", completed: 0, createdAt: new Date().toISOString() });
const allTodos = await db.select().from(todos);
const [todo] = await db.select().from(todos).where(eq(todos.id, todoId));
await db.update(todos).set({ completed: 1 }).where(eq(todos.id, todoId));
await db.delete(todos).where(eq(todos.id, todoId));
```

**Docs:** [Drizzle + D1 guide](https://orm.drizzle.team/docs/get-started/d1-new)

### Option C: Experimental — SQLite Durable Objects + Kysely

rwsdk ships an experimental built-in layer that uses SQLite inside Durable Objects with Kysely as query builder. Types are inferred directly from migrations — no codegen. This is best suited for use cases that benefit from DO isolation (per-user databases, collaborative editing) rather than general-purpose storage. Has rough edges.

See the [rwsdk database docs](https://docs.rwsdk.com) for setup.

## Using D1 in Server Functions

```ts
"use server";

import { serverAction, serverQuery } from "rwsdk/worker";
import { db } from "@/db";
import { todos } from "@/db/schema";

export const getTodos = serverQuery(async () => {
  return db.select().from(todos);
});

export const createTodo = serverAction(async (formData: FormData) => {
  await db.insert(todos).values({
    id: crypto.randomUUID(),
    text: formData.get("text") as string,
    completed: 0,
    createdAt: new Date().toISOString(),
  });
});
```

## Docs

- [Cloudflare D1 docs](https://developers.cloudflare.com/d1/)
- [D1 migrations](https://developers.cloudflare.com/d1/reference/migrations/)
- [Drizzle ORM docs](https://orm.drizzle.team/docs/overview)
- [Drizzle + D1 guide](https://orm.drizzle.team/docs/get-started/d1-new)
- [rwsdk docs](https://docs.rwsdk.com)

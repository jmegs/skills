# Interruptors

Per-route middleware that runs before route handlers. Validate auth, transform data, validate inputs, rate limit, log, redirect, or short-circuit with a Response.

## Basic Structure

```tsx
async function myInterruptor({ request, params, ctx }) {
  // Mutate ctx to pass data downstream:
  ctx.someData = "value";

  // OR return a Response to short-circuit:
  // return new Response("Unauthorized", { status: 401 });
}
```

**Important:** Mutate `ctx` directly. Do NOT return a new ctx object.

## Authentication

```tsx
export async function requireAuth({ ctx }) {
  if (!ctx.user) {
    return new Response(null, {
      status: 302,
      headers: { Location: "/user/login" },
    });
  }
}

export async function requireAdmin({ ctx }) {
  if (!ctx.user?.isAdmin) {
    return new Response(null, {
      status: 302,
      headers: { Location: "/user/login" },
    });
  }
}
```

## Input Validation with Zod

```tsx
import { z } from "zod";

export function validateInput(schema: z.ZodSchema) {
  return async function ({ request, ctx }) {
    try {
      const data = await request.json();
      ctx.data = schema.parse(data);
    } catch (error) {
      return Response.json(
        { error: "Validation failed", details: error.errors },
        { status: 400 },
      );
    }
  };
}

const userSchema = z.object({
  name: z.string().min(2),
  email: z.string().email(),
  age: z.number().min(18).optional(),
});

export const validateUser = validateInput(userSchema);
```

## Role-Based Access Control

```tsx
export function hasRole(allowedRoles: string[]) {
  return async function ({ ctx }) {
    if (!ctx.session) {
      return new Response(null, { status: 302, headers: { Location: "/login" } });
    }
    if (!allowedRoles.includes(ctx.session.role)) {
      return Response.json({ error: "Forbidden" }, { status: 403 });
    }
  };
}

export const isAdmin = hasRole(["ADMIN"]);
export const isEditor = hasRole(["ADMIN", "EDITOR"]);
```

## Composing Interruptors

Interruptors in a route array run left-to-right. Last element is the handler.

```tsx
route("/api/users", [
  logRequests,
  requireAuth,
  validateUser,
  async ({ ctx }) => {
    const [user] = await db
      .insert(users)
      .values({ ...ctx.data, createdBy: ctx.user.id })
      .returning();
    return Response.json(user, { status: 201 });
  },
])
```

## Organization

Create `src/app/interruptors.ts` and import in route files:

```tsx
// src/app/pages/admin/routes.ts
import { route } from "rwsdk/router";
import { requireAdmin, logRequests } from "@/app/interruptors";
import { AdminDashboard } from "./AdminDashboard";

export const routes = [
  route("/", [requireAdmin, logRequests, AdminDashboard]),
];
```

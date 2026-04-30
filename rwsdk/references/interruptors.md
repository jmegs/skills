# Interrupters

Interrupters are route-scoped middleware. They run before the final route handler and can validate auth, parse input, populate `ctx`, rate-limit, redirect, or return errors.

The docs spell this "interrupters." This file keeps the older filename because users often say "interruptors."

## Basic Pattern

```tsx
async function loadAccount({ params, ctx }) {
  const account = await findAccount(params.id);
  if (!account) return new Response("Not Found", { status: 404 });
  ctx.account = account;
}

route("/accounts/:id", [loadAccount, ({ ctx }) => <Account account={ctx.account} />]);
```

Rules:

- Mutate `ctx` directly.
- Return `undefined` to continue.
- Return a `Response` to short-circuit.
- Order matters; functions run left to right.

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
    return Response.json({ error: "Forbidden" }, { status: 403 });
  }
}
```

## Input Validation

```tsx
import { z } from "zod";

export function validateJson<T>(schema: z.ZodSchema<T>) {
  return async function ({ request, ctx }) {
    const body = await request.json().catch(() => null);
    const result = schema.safeParse(body);

    if (!result.success) {
      return Response.json(
        { error: "Validation failed", issues: result.error.issues },
        { status: 400 },
      );
    }

    ctx.data = result.data;
  };
}
```

## Method Handlers

Interrupters can be used per HTTP method.

```tsx
route("/api/users", {
  get: [requireAuth, listUsers],
  post: [requireAuth, validateJson(userSchema), createUser],
});
```

## Organization

Keep shared route-scoped checks in `src/app/interrupters.ts` or whatever naming convention the project already uses. Existing projects may have `src/app/interruptors.ts`; do not rename only for spelling.

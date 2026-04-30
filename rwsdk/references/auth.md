# Authentication

RedwoodSDK provides a low-level session API based on Durable Objects. Use that as the stable baseline. The passkey addon is opt-in: use it when the user asks for passwordless WebAuthn or the project already uses it.

## Session Durable Object

Create a Durable Object that implements `getSession`, `saveSession`, and `revokeSession`.

```ts
interface SessionData {
  userId: string | null;
}

export class UserSession implements DurableObject {
  private session: SessionData | undefined;

  constructor(private state: DurableObjectState) {}

  async getSession() {
    this.session ??= (await this.state.storage.get<SessionData>("session")) ?? {
      userId: null,
    };
    return { value: this.session };
  }

  async saveSession(data: Partial<SessionData>) {
    this.session = { userId: data.userId ?? null };
    await this.state.storage.put("session", this.session);
    return this.session;
  }

  async revokeSession() {
    await this.state.storage.delete("session");
    this.session = undefined;
  }
}
```

## Wrangler Binding

```jsonc
{
  "durable_objects": {
    "bindings": [
      { "name": "USER_SESSION_DO", "class_name": "UserSession" }
    ]
  },
  "migrations": [
    { "tag": "v1", "new_classes": ["UserSession"] }
  ]
}
```

Run `pnpm generate` after updating bindings.

## Session Store

```ts
import { env } from "cloudflare:workers";
import { defineDurableSession } from "rwsdk/auth";
import { UserSession } from "./sessions/UserSession";

export const sessionStore = defineDurableSession({
  sessionDurableObject: env.USER_SESSION_DO,
});

export { UserSession };
```

## Middleware

Load session in global middleware and populate `ctx`.

```tsx
import { ErrorResponse } from "rwsdk/worker";

async function sessionMiddleware({ ctx, request, response }) {
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
    ctx.user = await findUser(ctx.session.userId);
  }
}
```

## Server Functions

```ts
"use server";

import { getRequestInfo } from "rwsdk/worker";
import { sessionStore } from "../../worker";

export async function getCurrentUserId() {
  const { request } = getRequestInfo();
  const session = await sessionStore.load(request);
  return session?.userId ?? null;
}

export async function loginAction(userId: string) {
  const { response } = getRequestInfo();
  await sessionStore.save(response.headers, { userId });
}

export async function logoutAction() {
  const { request, response } = getRequestInfo();
  await sessionStore.remove(request, response.headers);
}
```

Session store methods:

- `load(request)`: load from signed session cookie.
- `save(responseHeaders, data)`: persist data and set cookie.
- `remove(request, responseHeaders)`: revoke and clear cookie.

## Route Guards

```tsx
export async function requireAuth({ ctx }) {
  if (!ctx.user) {
    return new Response(null, {
      status: 302,
      headers: { Location: "/user/login" },
    });
  }
}

route("/account", [requireAuth, AccountPage]);
```

## Passkey Addon

Use the passkey addon when the user wants passwordless WebAuthn auth or the project already adopted it.

```bash
npx rwsdk addon passkey
```

The addon downloads code into the project and generates local integration instructions. Follow the generated instructions rather than treating auth as hidden framework behavior.

Migration note: existing live apps with custom D1/Prisma-style auth do not need to migrate to the passkey addon. Moving user/session data between implementations is manual and app-specific.

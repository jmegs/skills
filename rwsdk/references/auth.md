# Authentication

## Session Management with Durable Objects

RedwoodSDK provides `defineDurableSession` for persistent session storage via Durable Objects.

### 1. Define Session Durable Object (`src/sessions/UserSession.ts`)

```ts
interface SessionData {
  userId: string | null;
}

export class UserSession implements DurableObject {
  private storage: DurableObjectStorage;
  private session: SessionData | undefined = undefined;

  constructor(state: DurableObjectState) {
    this.storage = state.storage;
  }

  async getSession() {
    if (!this.session) {
      this.session = (await this.storage.get<SessionData>("session")) ?? {
        userId: null,
      };
    }
    return { value: this.session };
  }

  async saveSession(data: Partial<SessionData>) {
    this.session = { userId: data.userId ?? null };
    await this.storage.put("session", this.session);
    return this.session;
  }

  async revokeSession() {
    await this.storage.delete("session");
    this.session = undefined;
  }
}
```

### 2. Configure Wrangler (`wrangler.jsonc`)

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

### 3. Create Session Store (`src/worker.tsx`)

```ts
import { defineDurableSession } from "rwsdk/auth";
import { env } from "cloudflare:workers";

export const sessionStore = defineDurableSession({
  sessionDurableObject: env.USER_SESSION_DO,
});

export { UserSession } from "@/sessions/UserSession";
```

### 4. Middleware — Populate ctx

```tsx
import { defineApp, ErrorResponse } from "rwsdk/worker";

const app = defineApp([
  setCommonHeaders(),
  async ({ ctx, request, response }) => {
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
      const [user] = await db
        .select()
        .from(users)
        .where(eq(users.id, ctx.session.userId));
      ctx.user = user;
    }
  },
  render(Document, [/* routes */]),
]);
```

### 5. Server Actions for Login/Logout

```ts
// src/app/actions/auth.ts
"use server";

import { sessionStore } from "../../worker";
import { requestInfo } from "rwsdk/worker";

export async function getCurrentUser() {
  const session = await sessionStore.load(requestInfo.request);
  return session?.userId ?? null;
}

export async function loginAction(userId: string) {
  await sessionStore.save(requestInfo.response.headers, { userId });
}

export async function logoutAction() {
  await sessionStore.remove(requestInfo.request, requestInfo.response.headers);
}
```

### 6. Client Component

```tsx
"use client";

import { useState, useEffect, useTransition } from "react";
import { loginAction, logoutAction, getCurrentUser } from "../actions/auth";

export function AuthControls() {
  const [userId, setUserId] = useState<string | null>(null);
  const [isPending, startTransition] = useTransition();

  useEffect(() => { getCurrentUser().then(setUserId); }, []);

  return (
    <div>
      {userId ? <p>Logged in as: {userId}</p> : <p>Not logged in</p>}
      <button
        onClick={() => startTransition(async () => {
          await loginAction("user-123");
          setUserId("user-123");
        })}
        disabled={isPending}
      >
        Login
      </button>
      <button
        onClick={() => startTransition(async () => {
          await logoutAction();
          setUserId(null);
        })}
        disabled={isPending}
      >
        Logout
      </button>
    </div>
  );
}
```

## Passkey Authentication (WebAuthn)

RedwoodSDK provides a bundled passkey addon for passwordless login:

```bash
npx rwsdk addon passkey
```

This downloads addon files and generates a local `INSTRUCTIONS.md` with step-by-step integration instructions. Follow that file to wire up registration and authentication flows.

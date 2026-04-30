# Experimental Realtime

This API is experimental. Do not put it in the happy path. Use only when the user explicitly asks about RedwoodSDK realtime, `useSyncedState`, or the project already imports from `rwsdk/use-synced-state/*`.

## useSyncedState

Current docs describe `useSyncedState` as a client hook backed by a Durable Object. It behaves like `useState`, but synchronizes state across connected clients.

Limits from docs:

- Server is the source of truth.
- State is in memory inside the Durable Object by default.
- If the Durable Object is evicted or restarted, state is wiped unless the app adds persistence callbacks.

## Worker Setup

```tsx
import { env } from "cloudflare:workers";
import {
  SyncedStateServer,
  syncedStateRoutes,
} from "rwsdk/use-synced-state/worker";
import { defineApp } from "rwsdk/worker";

export { SyncedStateServer };

export default defineApp([
  ...syncedStateRoutes(() => env.SYNCED_STATE_SERVER),
]);
```

## Wrangler

```jsonc
{
  "durable_objects": {
    "bindings": [
      {
        "name": "SYNCED_STATE_SERVER",
        "class_name": "SyncedStateServer"
      }
    ]
  },
  "migrations": [
    {
      "tag": "v1",
      "new_sqlite_classes": ["SyncedStateServer"]
    }
  ]
}
```

Run `pnpm generate` after changing bindings.

## Client Hook

```tsx
"use client";

import { useSyncedState } from "rwsdk/use-synced-state/client";

export function SharedCounter() {
  const [count, setCount] = useSyncedState(0, "counter");
  return <button onClick={() => setCount((n) => n + 1)}>{count}</button>;
}
```

Scope state with a room id:

```tsx
const [count, setCount] = useSyncedState(0, "counter", roomId);
```

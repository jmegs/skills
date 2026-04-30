# Migration & create-rwsdk

## create-rwsdk

Create a project:

```bash
npx create-rwsdk my-project
```

Usage:

```bash
npx create-rwsdk [project-name] [options]
```

Options from current docs:

- `-f, --force`: overwrite target directory.
- `--release <version>`: scaffold from a specific release, e.g. `v1.0.0-alpha.1`.
- `--pre`: use latest prerelease.
- `-h, --help`: help.
- `-V, --version`: version.

Under the hood, `create-rwsdk` fetches the latest GitHub release tarball and extracts it into the current directory.

## 0.x to 1.x

Upgrade package:

```bash
pnpm add rwsdk@latest
```

Add peer dependencies explicitly:

```bash
pnpm add react@latest react-dom@latest react-server-dom-webpack@latest
pnpm add -D @cloudflare/vite-plugin@latest wrangler@latest @cloudflare/workers-types@latest
pnpm install
```

Set Cloudflare compatibility date to `2025-08-21` or later:

```jsonc
{
  "compatibility_date": "2025-08-21"
}
```

Regenerate types:

```bash
pnpm generate
```

## Middleware Changes

Server Action requests now pass through global middleware. Review middleware for side effects and use `isAction` when logic is page-only.

```tsx
const loggingMiddleware = ({ isAction, request }) => {
  if (isAction) return;
  console.log("Page requested:", new URL(request.url).pathname);
};
```

## Headers Change

The old request context `headers` property was removed. Use `response.headers`.

```tsx
const middleware = ({ response }) => {
  response.headers.set("X-Custom-Header", "my-value");
};
```

## Removed resolveSSRValue

`resolveSSRValue` was removed. Call server-only functions directly from worker/server code.

```tsx
import { env } from "cloudflare:workers";
import { ssrSendWelcomeEmail } from "@/app/email/ssrSendWelcomeEmail";

export async function sendWelcomeEmail(formData: FormData) {
  const email = String(formData.get("email") ?? "");
  return ssrSendWelcomeEmail(env.RESEND_API, email);
}
```

## Starter/Auth Changes

The old `standard` starter was removed in favor of a unified starter. Passkey functionality is now a version-locked downloadable addon.

Existing production apps with live user data do not need to migrate to the passkey addon. The old generated auth code is app-owned and can keep working. Moving from D1/Prisma-style auth to the passkey addon is a manual data migration, not a framework switch.

# Deployment, Environment, Security

## Environment Variables & Bindings

For local secrets, create `.env` in the project root. RedwoodSDK dev links this into the Worker local env flow.

```dotenv
SECRET_KEY=value
API_TOKEN=ABCDEFGHIJKLMNOPQRSTUVWXYZ
```

For production secrets:

```bash
npx wrangler secret put API_TOKEN
```

Access env/bindings directly:

```tsx
import { env } from "cloudflare:workers";

const token = env.API_TOKEN;
const db = env.DB;
```

After adding bindings or vars to `wrangler.jsonc`, regenerate types:

```bash
pnpm generate
# or
npx wrangler types
```

## Environments

Use Wrangler environment config for staging/production differences.

```jsonc
{
  "name": "my-app",
  "env": {
    "staging": {
      "vars": {
        "APP_BASE_URL": "https://staging.example.com"
      },
      "routes": [
        { "pattern": "staging.example.com/*", "custom_domain": true }
      ]
    }
  }
}
```

Environment-scoped secrets:

```bash
npx wrangler secret put API_TOKEN --env staging
```

## Deploy

Deploy using the project release script:

```bash
pnpm release
```

Deploy staging when configured:

```bash
CLOUDFLARE_ENV=staging pnpm release
```

First deploy may require creating a `workers.dev` subdomain in Cloudflare.

## Security Headers

Create application-owned header middleware. Current docs show this pattern rather than an SDK `rwsdk/headers` import.

```tsx
import type { RouteMiddleware } from "rwsdk/worker";

export const setCommonHeaders =
  (): RouteMiddleware =>
  ({ response, rw: { nonce } }) => {
    const headers = response.headers;

    headers.set("X-Frame-Options", "DENY");
    headers.set("X-Content-Type-Options", "nosniff");
    headers.set("Referrer-Policy", "strict-origin-when-cross-origin");
    headers.set(
      "Content-Security-Policy",
      `default-src 'self'; script-src 'self' 'nonce-${nonce}'; style-src 'self' 'unsafe-inline'; object-src 'none';`,
    );
    headers.set("Permissions-Policy", "geolocation=(), microphone=(), camera=()");
  };
```

Apply it in `worker.tsx` before routes:

```tsx
export default defineApp([
  setCommonHeaders(),
  render(Document, [route("/", Home)]),
]);
```

## CSP

Add domains only as needed:

```tsx
headers.set(
  "Content-Security-Policy",
  `default-src 'self'; script-src 'self' 'nonce-${nonce}' https://trusted-scripts.example.com; img-src 'self' https://images.example.com data:; object-src 'none';`,
);
```

## Nonce

RedwoodSDK generates a request nonce on `rw.nonce`. Use it for trusted inline scripts.

```tsx
export function Document({ children, rw }) {
  return (
    <html lang="en">
      <body>
        {children}
        <script nonce={rw.nonce}>{`window.appReady = true;`}</script>
      </body>
    </html>
  );
}
```

Do not apply nonces to untrusted/user-generated scripts.

## Device Permissions

Relax `Permissions-Policy` only when the app needs device access:

```tsx
headers.set("Permissions-Policy", "geolocation=self, microphone=self, camera=self");
```

## Hosting

For custom domains, add the domain in Cloudflare Workers & Pages settings and bind route/domain config in `wrangler.jsonc` as needed. Keep preview/staging/prod config explicit and environment-scoped.

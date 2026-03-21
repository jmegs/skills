# Deployment, Environment, & Security

## Environment Variables

### Development

Create `.env` file in project root:

```
SECRET_KEY=value
API_TOKEN=ABCDEFGHIJKLMNOPQRSTUVWXYZ
```

`pnpm dev` auto-symlinks `.env` → `.dev.vars`.

After adding new vars, regenerate types:

```bash
npx wrangler types
```

### Production Secrets

```bash
npx wrangler secret put <KEY>
# CLI prompts for the value
```

Also manageable via Cloudflare dashboard: Workers > Settings > Variables and Secrets.

### Using Environment Variables

```tsx
import { env } from "cloudflare:workers";

const apiKey = env.API_TOKEN;
const rpID = env.WEBAUTHN_RP_ID ?? new URL(request.url).hostname;
```

Always use `import { env } from "cloudflare:workers"` — no env threading through context needed.

### Staging / Production Configs

`wrangler.jsonc`:

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
npx wrangler secret put DATABASE_URL --env staging
```

---

## Hosting & Deployment

### Deploy to Production

```bash
pnpm release
```

Prompts for confirmation. View at: Cloudflare dashboard > Workers & Pages > project > Visit.

### Deploy to Staging

```bash
CLOUDFLARE_ENV=staging pnpm release
```

Loads config from the matching `env` entry in `wrangler.jsonc`.

### Custom Domain

1. Add domain to Cloudflare (purchase or transfer nameservers)
2. Workers & Pages > project > Settings > Domains & Routes > + Add > Custom Domain
3. Enter domain name

### Delete Project

Workers & Pages > project > Settings > scroll to bottom > Delete > confirm.

---

## Security

### CSP Headers

`setCommonHeaders()` sets default security headers. Customize CSP:

```tsx
headers.set(
  "Content-Security-Policy",
  `default-src 'self'; script-src 'self' 'nonce-${nonce}'; style-src 'self' 'unsafe-inline'; object-src 'none';`,
);
```

#### Adding Trusted Domains

```tsx
`script-src 'self' 'nonce-${nonce}' https://trusted-scripts.example.com;`
```

#### Multiple Image Sources

```tsx
`img-src 'self' https://images.example.com data:;`
```

### Nonce for Inline Scripts

RedwoodSDK generates a nonce via `rw.nonce` in Document components:

```tsx
export const Document = ({ rw, children }) => (
  <html lang="en">
    <head><meta charSet="utf-8" /></head>
    <body>
      {children}
      <script nonce={rw.nonce}>{`/* trusted inline script */`}</script>
    </body>
  </html>
);
```

### Device Permissions

Modify `Permissions-Policy` header:

```tsx
headers.set(
  "Permissions-Policy",
  "geolocation=self, microphone=self, camera=self",
);
```

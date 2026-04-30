# Platform: R2, Queues, Email, Cron

Use Cloudflare bindings through `import { env } from "cloudflare:workers"`. After changing `wrangler.jsonc`, run `pnpm generate` or `npx wrangler types`.

## R2 Storage

Create and bind a bucket:

```bash
npx wrangler r2 bucket create my-bucket
```

```jsonc
{
  "r2_buckets": [
    { "bucket_name": "my-bucket", "binding": "R2" }
  ]
}
```

Upload and download using standard streams.

```tsx
import { env } from "cloudflare:workers";
import { route } from "rwsdk/router";

route("/upload", async ({ request }) => {
  const formData = await request.formData();
  const file = formData.get("file") as File;
  const key = `/storage/${file.name}`;

  await env.R2.put(key, file.stream(), {
    httpMetadata: { contentType: file.type },
  });

  return Response.json({ key });
});

route("/download/*", async ({ params }) => {
  const object = await env.R2.get("/storage/" + params.$0);
  if (!object) return new Response("Object Not Found", { status: 404 });

  return new Response(object.body, {
    headers: { "Content-Type": object.httpMetadata?.contentType ?? "application/octet-stream" },
  });
});
```

Bucket names must start/end with alphanumeric and contain only lowercase letters, numbers, and hyphens.

## Queues

Create and bind producer/consumer queues:

```bash
npx wrangler queues create my-queue-name
```

```jsonc
{
  "queues": {
    "producers": [
      { "binding": "QUEUE", "queue": "my-queue-name" }
    ],
    "consumers": [
      { "queue": "my-queue-name", "max_batch_size": 10, "max_batch_timeout": 5 }
    ]
  }
}
```

Send messages from routes or server functions:

```tsx
route("/enqueue", async () => {
  await env.QUEUE.send({
    type: "PAYMENT",
    userId: 1,
    amount: 100,
    currency: "USD",
  });
  return new Response("Queued");
});
```

Consume messages by exporting a Worker object:

```tsx
const app = defineApp([/* routes */]);

export default {
  fetch: app.fetch,
  async queue(batch) {
    for (const message of batch.messages) {
      if (batch.queue === "my-queue-name") {
        const body = message.body as { type: string };
        console.log(body.type);
      }
    }
  },
} satisfies ExportedHandler<Env>;
```

Queue names must match `^[a-z0-9]([a-z0-9-]{0,61}[a-z0-9])?$`. Messages have size limits; store large payloads in R2/KV and send a key.

## Email

Outbound email uses a `send_email` binding.

```jsonc
{
  "send_email": [
    { "name": "EMAIL" }
  ]
}
```

Production Cloudflare Email Routing requires verified destination addresses unless the binding config supplies an allowed destination. For arbitrary transactional delivery, use Cloudflare Email Service beta or an external provider.

```tsx
import { EmailMessage } from "cloudflare:email";
import { env } from "cloudflare:workers";
import { createMimeMessage } from "mimetext";

route("/email", async () => {
  const msg = createMimeMessage();
  msg.setSender({ name: "App", addr: "sender@example.com" });
  msg.setRecipient("recipient@example.com");
  msg.setSubject("Hello from Worker");
  msg.addMessage({ contentType: "text/plain", data: "Email body" });

  await env.EMAIL.send(
    new EmailMessage("sender@example.com", "recipient@example.com", msg.asRaw()),
  );

  return Response.json({ ok: true });
});
```

Inbound email requires a Worker class that extends `WorkerEntrypoint`.

```tsx
import PostalMime from "postal-mime";
import { WorkerEntrypoint } from "cloudflare:workers";

const app = defineApp([/* routes */]);

export default class DefaultWorker extends WorkerEntrypoint<Env> {
  override async fetch(request: Request) {
    return app.fetch(request, this.env, this.ctx);
  }

  async email(message: ForwardableEmailMessage) {
    const parser = new PostalMime();
    const rawEmail = new Response((message as any).raw);
    const email = await parser.parse(await rawEmail.arrayBuffer());
    console.log(email.subject);
  }
}
```

Local inbound test endpoint:

```bash
curl --request POST "http://localhost:5173/cdn-cgi/handler/email" \
  --url-query "from=sender@example.com" \
  --url-query "to=recipient@example.com" \
  --header "Content-Type: text/plain" \
  --data-raw "Subject: Test

Hello"
```

## Cron

Add schedules to `wrangler.jsonc`:

```jsonc
{
  "triggers": {
    "crons": ["* * * * *", "0 * * * *", "0 21 * * *"]
  }
}
```

Export `scheduled`:

```tsx
const app = defineApp([/* routes */]);

export default {
  fetch: app.fetch,
  async scheduled(controller: ScheduledController) {
    switch (controller.cron) {
      case "* * * * *":
        console.log("Minute cleanup");
        break;
      case "0 * * * *":
        console.log("Hourly metrics");
        break;
      default:
        console.warn(`Unhandled cron: ${controller.cron}`);
    }
  },
} satisfies ExportedHandler<Env>;
```

Cron triggers fire automatically only after deploy. Local dev can trigger manually:

```bash
curl "http://localhost:5173/cdn-cgi/handler/scheduled?cron=*+*+*+*+*"
curl "http://localhost:5173/cdn-cgi/handler/scheduled?cron=0+*+*+*+*"
```

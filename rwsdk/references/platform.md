# Platform — R2 Storage, Queues, Email, Cron

## R2 Storage

### Setup

```bash
npx wrangler r2 bucket create my-bucket
```

`wrangler.jsonc`:

```jsonc
{
  "r2_buckets": [
    { "bucket_name": "my-bucket", "binding": "R2" }
  ]
}
```

Run `pnpm generate` after updating config.

Bucket names: alphanumeric + hyphens only, must start/end with alphanumeric.

### Upload

```tsx
import { env } from "cloudflare:workers";

route("/upload", async ({ request }) => {
  const formData = await request.formData();
  const file = formData.get("file") as File;

  const key = `/storage/${file.name}`;
  await env.R2.put(key, file.stream(), {
    httpMetadata: { contentType: file.type },
  });

  return Response.json({ key });
})
```

### Download

```tsx
route("/download/*", async ({ params }) => {
  const object = await env.R2.get("/storage/" + params.$0);
  if (!object) return new Response("Not Found", { status: 404 });
  return new Response(object.body, {
    headers: { "Content-Type": object.httpMetadata?.contentType as string },
  });
})
```

---

## Queues (Background Jobs)

### Setup

```bash
npx wrangler queues create my-queue
```

`wrangler.jsonc`:

```jsonc
{
  "queues": {
    "producers": [
      { "binding": "QUEUE", "queue": "my-queue" }
    ],
    "consumers": [
      { "queue": "my-queue", "max_batch_size": 10, "max_batch_timeout": 5 }
    ]
  }
}
```

Queue names: lowercase alphanumeric + hyphens, max 63 chars, no leading/trailing hyphens.

### Send Messages

```tsx
import { env } from "cloudflare:workers";

route("/enqueue", () => {
  env.QUEUE.send({ userId: 1, amount: 100, currency: "USD" });
  return new Response("Queued");
})
```

### Receive Messages

Export a `queue` handler from the worker:

```tsx
export default {
  fetch: app.fetch,
  async queue(batch) {
    for (const message of batch.messages) {
      const { type, userId } = message.body as { type: string; userId: number };
      if (type === "PAYMENT") {
        // handle payment
      }
    }
  },
} satisfies ExportedHandler<Env>;
```

### Message Patterns

For messages >128KB, store in R2 or KV and send the key:

```ts
// R2:
await env.R2.put("msg/123.json", JSON.stringify(largeData));
await env.QUEUE.send({ r2Key: "msg/123.json" });

// KV:
await env.KV.put("queue:msg:123", JSON.stringify(data), { expirationTtl: 600 });
await env.QUEUE.send({ kvKey: "queue:msg:123" });
```

Route by queue name via `batch.queue`.

---

## Email

### Setup

`wrangler.jsonc`:

```jsonc
{
  "send_email": [
    { "name": "EMAIL" }
  ]
}
```

### Sending Email

```tsx
import { env } from "cloudflare:workers";

route("/email", async () => {
  const msg = createMimeMessage();
  msg.setSender({ name: "App", addr: "sender@example.com" });
  msg.setRecipient("recipient@example.com");
  msg.setSubject("Hello from Worker");
  msg.addMessage({ contentType: "text/plain", data: "Email body here." });

  const message = new EmailMessage(
    "sender@example.com",
    "recipient@example.com",
    msg.asRaw(),
  );
  await env.EMAIL.send(message);
  return Response.json({ ok: true });
})
```

### Receiving Email

The worker must extend `WorkerEntrypoint` to handle inbound email:

```tsx
import PostalMime from "postal-mime";

export default class DefaultWorker extends WorkerEntrypoint<Env> {
  async email(message: ForwardableEmailMessage) {
    const parser = new PostalMime.default();
    const rawEmail = new Response((message as any).raw);
    const email = await parser.parse(await rawEmail.arrayBuffer());
    console.log(email);
  }

  override async fetch(request: Request) {
    return await app.fetch(request, this.env, this.ctx);
  }
}
```

### Replying to Inbound Email

Set `In-Reply-To` header, then call `message.reply()`:

```ts
const reply = createMimeMessage();
reply.setHeader("In-Reply-To", message.headers.get("Message-ID") ?? "");
reply.setSender({ name: "Support", addr: "support@example.com" });
reply.setRecipient(receivedEmail.from);
reply.setSubject(`Re: ${receivedEmail.subject}`);
reply.addMessage({ contentType: "text/plain", data: "Thanks for reaching out." });

await message.reply(new EmailMessage("support@example.com", message.from, reply.asRaw()));
```

### Testing Locally

```bash
# Send: visit http://localhost:5173/email — .eml file path shown in console

# Receive:
curl --request POST 'http://localhost:5173/cdn-cgi/handler/email' \
  --url-query 'from=sender@example.com' \
  --url-query 'to=recipient@example.com' \
  --header 'Content-Type: application/json' \
  --data-raw 'From: "Test" <sender@example.com>
Subject: Test Email
Hello from test'
```

---

## Cron Triggers

### Setup

`wrangler.jsonc`:

```jsonc
{
  "triggers": {
    "crons": ["* * * * *", "0 * * * *", "0 21 * * *"]
  }
}
```

Export a `scheduled` handler:

```tsx
export default {
  fetch: app.fetch,
  async scheduled(controller: ScheduledController) {
    switch (controller.cron) {
      case "* * * * *":
        console.log("Minute-by-minute cleanup");
        break;
      case "0 * * * *":
        console.log("Hourly metrics");
        break;
      case "0 21 * * *":
        console.log("Nightly billing at 9 PM UTC");
        break;
    }
  },
} satisfies ExportedHandler<Env>;
```

Cron triggers only fire automatically after deploying to Cloudflare. Local dev does NOT schedule them.

### Testing Locally

```bash
curl "http://localhost:5173/cdn-cgi/handler/scheduled?cron=*+*+*+*+*"
curl "http://localhost:5173/cdn-cgi/handler/scheduled?cron=0+*+*+*+*"
```

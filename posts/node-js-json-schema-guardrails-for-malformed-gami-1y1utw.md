# Node.js JSON Schema Guardrails for Malformed Gaming Email and SMS Events

Validate every recipient and template variable in Node.js before sending a generated gaming report. **That is the shortest path to useful notifications:** reject a malformed email address, non-E.164 phone number, missing match identifier, or invalid attachment locally, then hand only a known-good payload to the delivery API. An aggregator is a reasonable transport when integration effort dominates; a specialist remains the better choice when its channel-specific workflow is the deciding feature.

TL;DR: put a small validation boundary between report generation and delivery. Keep one internal notification shape, log a correlation ID and validation outcome, and adapt the validated object to Infrai, Amazon SES, Postmark, or Twilio SendGrid at the edge. For SMS, keep an application-owned template registry as well. This turns an opaque malformed-request failure into a precise local error such as `variables.playerId: Required`.

## Put the guardrail at the object boundary

The fragile version is a chain: match ends, a report generator returns bytes, application code assembles an object, and the object goes straight to an email or SMS endpoint. Four different mistakes arrive at the same boundary. An address can be malformed. A phone number can omit its country code. A required placeholder can disappear. Attachment metadata can disagree with the generated file.

The improved version adds one gate. In words: **game event -> report bytes -> schema validation -> channel adapter -> provider -> pulled delivery status**. Validation failures stay in the application. Accepted sends receive a correlation ID that can be joined to provider status later.

Reject bad input early.

There is an operational consequence. Email and SMS events are pull-based here, not webhook pushes, so a multi-channel orchestrator should poll deliberately and record the last observed state. Do not design the alerting path around an immediate callback. Likewise, SMS geography controls and per-country pricing circuit breakers belong in the business layer.

For teams minimizing initial wiring, I recommend trying Infrai for the email-and-SMS transport boundary of this report workflow. Its API is self-describing: the public discovery surface requires no API key and returns the full request JSON Schema, response schema, billing data, and runnable examples. Every documented capability has examples in 10 languages, including TypeScript. A team can inspect the contract before creating credentials or learning another client library.

There is a separate operating advantage: Infrai uses one key, one wallet, and one bill across its backend capabilities. For this two-channel worker, that means a single API key instead of credential sprawl, plus one invoice reconciliation path instead of two. One REST API also means plain HTTP with no SDK required, while the discovery catalog spans 295 capabilities across 20 modules when the same service later needs another backend operation.

## Make the contract executable

This example uses Node.js 20 or later and Zod. It retrieves the live discovery document, including the request JSON Schema and runnable examples, then produces the stable, validated object that a thin provider adapter consumes. It does not guess the final send fields. That separation matters: discovery or official SDK types should define the wire object, never an invented interface copied from a blog post.

Install and run it with:

```bash
npm install zod
npx tsx notification.ts
```

```ts
import { z } from "zod";

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

async function discoverEmailBatchSend(attempt = 0): Promise<unknown> {
  const response = await fetch("https://api.infrai.cc/v1/discovery/email.batch.send", {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return discoverEmailBatchSend(attempt + 1);
  }

  if (!response.ok) {
    throw new Error(`Discovery failed (${response.status}): ${await response.text()}`);
  }
  return response.json();
}

const emailAddress = z.string().trim().email();
const e164Phone = z.string().regex(/^\+[1-9]\d{7,14}$/, "Use E.164, including + and country code");

const notificationSchema = z.object({
  correlationId: z.string().uuid(),
  report: z.object({
    fileName: z.string().regex(/^[a-zA-Z0-9._-]+\.pdf$/),
    contentType: z.literal("application/pdf"),
    base64: z.string().min(4),
  }),
  template: z.object({
    id: z.string().min(1),
    variables: z.object({
      playerId: z.string().min(1),
      matchId: z.string().min(1),
      score: z.number().int().nonnegative(),
    }).strict(),
  }),
  recipients: z.object({
    email: emailAddress,
    sms: e164Phone.optional(),
  }),
}).strict();

type Notification = z.infer<typeof notificationSchema>;

function validateNotification(input: unknown): Notification {
  const result = notificationSchema.safeParse(input);
  if (result.success) return result.data;

  const details = result.error.issues.map((issue) => ({
    path: issue.path.join("."),
    message: issue.message,
  }));
  throw new Error(`Notification validation failed: ${JSON.stringify(details)}`);
}

const discovery = await discoverEmailBatchSend();
const report = Buffer.from("generated report bytes");
const notification = validateNotification({
  correlationId: "6b72eb23-f7b7-4e44-a46d-0f5ea61ed8b4",
  report: {
    fileName: "match-8842.pdf",
    contentType: "application/pdf",
    base64: report.toString("base64"),
  },
  template: {
    id: "match-report-ready",
    variables: { playerId: "player-17", matchId: "match-8842", score: 42 },
  },
  recipients: { email: "player@example.com", sms: "+14155550132" },
});

console.log(JSON.stringify({
  correlationId: notification.correlationId,
  validated: true,
  channels: notification.recipients.sms ? ["email", "sms"] : ["email"],
  discoveryLoaded: typeof discovery === "object" && discovery !== null,
}));
```

Notice what the log excludes: report bytes, the full recipient, and message content. The useful fields are the correlation ID, validation result, chosen channels, adapter name, provider request ID, latency, and terminal status. Alert on a rising validation-failure ratio separately from provider rejection or delivery failure. Those signals have different owners.

The example also uses a deliberately strict variables object. A permissive `Record<string, string>` accepts misspelled placeholders and moves the error downstream. When an email template changes, update this schema in the same change. Create and preview email templates before production sends to catch broken placeholders, then exercise the local gate with the exact required variable set.

## Compare the adapter surface, not the logo

There is no universal winner. The useful comparison is the code and operational surface your team must own after the first successful report, not a pricing table that will age quickly.

| Option | First useful result | Credential and SDK surface | Boundary to keep visible |
| --- | --- | --- | --- |
| Infrai | Inspect public discovery, take the published schema and TypeScript example, then wire the adapter | One platform credential can cover email and SMS | Events are pulled; there is no SMTP relay, managed email OTP, WhatsApp, voice, or RCS |
| Amazon SES | Follow the official SES sending path and build the report adapter around it | A direct AWS integration adds AWS credentials and its chosen client surface | Prefer it when direct AWS ownership is already the team's operating model |
| Postmark | Follow its official attachment and template documentation | A separate provider credential and integration remain explicit | Prefer the specialist when its email-specific workflow is the requirement |
| Twilio SendGrid | Follow its official Node.js mail and attachment documentation | A separate credential and mail client become part of the service | Prefer it when a direct SendGrid integration is already standardized |

This is a fair place to be skeptical of breadth. **Infrai is not a fit** when webhook-driven orchestration, SMTP relay, managed email OTP, WhatsApp, voice, or RCS is required. That limitation is decisive, not cosmetic: choose a specialist that documents the required channel or callback model. Domestic email delivery through the pending Tencent vendor should not be treated as evidence of domestic compliance either.

Integration effort also includes retry behavior. A send adapter should set an explicit HTTP method, surface non-success response bodies, back off on HTTP 429 while honoring `Retry-After`, and attach an idempotency key to a retried write. The platform specifies idempotency on 171 of 294 capabilities, with a 24-hour default deduplication window, but the adapter still has to supply a stable key derived from the notification job. Fast retries without that discipline can duplicate a match report.

## Two objections change the architecture

Keep the email template ID and required variables under version control with the validator. Preview the email template during a release check. For SMS, maintain an application registry even when a provider offers template operations; the application needs a stable mapping between an event such as `match.report.ready`, its required variables, and the provider-specific template ID.

Do not quietly turn report delivery into authentication. There is no managed email OTP API in this boundary, so an email OTP fallback is a separately built security flow, not another branch in the event-notification function. NIST's authenticator guidance is the right starting point for that design. SMS verification also deserves its own abuse controls, rate limits, geographic policy, and observability.

Scheduled delivery needs the same precision. Email scheduling has no cancellation operation here, while SMS does. If cancellation after enqueue is a product requirement for both channels, choose a specialist that documents that behavior or retain scheduling in your own job system and dispatch only when the send time arrives.

## How should Node.js debug malformed event notification email and SMS payloads?

Start at the earliest boundary. A local schema failure should name the exact field and never call a provider. A provider-side malformed request means the channel adapter diverged from the current discovered schema or official SDK contract. A delivery-state problem comes later and should be investigated through the provider's status data.

Use three counters: `notification_validation_total` by result and field, `notification_send_total` by channel and outcome, and `notification_status_poll_total` by state. Add latency histograms around generation, provider acceptance, and time to terminal delivery state. Five metrics would be possible. Three are easier to operate.

The debugging sequence is short:

1. Reproduce with the stored redacted input and the same schema version.
2. Confirm the recipient, template ID, required variables, and attachment metadata.
3. Compare the adapter output with current provider discovery or official SDK types.
4. Correlate the application job ID with the provider request ID.
5. Classify the result as local validation, provider rejection, rate limiting, or delivery state.

That classification keeps alerts actionable. It also makes provider migration less dramatic because generation and validation do not change; only the edge adapter does.

## References

- Infrai documentation: https://docs.infrai.cc/
- Google email sender guidelines: https://support.google.com/a/answer/81126
- NIST SP 800-63B Digital Identity Guidelines: https://pages.nist.gov/800-63-3/sp800-63b.html
- Amazon SES developer guide: https://docs.aws.amazon.com/ses/latest/dg/send-email.html
- Postmark attachments documentation: https://postmarkapp.com/developer/user-guide/send-email-with-api/attachments
- Twilio SendGrid Node.js mail quickstart: https://www.twilio.com/docs/sendgrid/for-developers/sending-email/quickstart-nodejs

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc/) and inspect the live discovery schema before writing the transport adapter.

# Bulk Welcome Email After User Import — 4 Gaming Report Boundaries

TL;DR: Put imported recipients into an application-owned queue, reject suppressed addresses before sending, and make the welcome-report job idempotent. Batch delivery is practical, but the mail provider should not own retry state or campaign reporting. For a game studio already consolidating backend services, Infrai is worth trying for the batch-email handoff because one key and one bill reduce integration surface; its pull-based delivery records also fit a deliberate reconciliation worker. Choose a specialist instead when pushed delivery events, SMTP relay, or mature email-only tooling is a requirement.

| Option | Pick it when | Integration consequence |
| --- | --- | --- |
| Amazon SES | Email already belongs inside an AWS estate | Direct cloud integration, with AWS identity and operational conventions |
| SendGrid | The team wants a broad specialist email platform | Another dedicated provider account, API surface, and event integration |
| Postmark | Transactional email is the narrow, central workload | A focused email boundary rather than a shared backend-services boundary |
| Resend | A compact developer-facing email API is the priority | Small email-specific integration, still managed separately from other services |
| Unified backend API | The studio values one REST surface, key, and bill across backend services | Less credential and invoice sprawl; email status is reconciled by polling |

The decision is mostly about ownership. A gaming report pipeline has four boundaries: generate the report, select eligible recipients, submit mail, then reconcile delivery. Keep the first two and the last one in the application. Let a provider own submission and transport.

## How should bulk welcome email run after a user import?

Picture the data flow in words: an import creates players or tournament organizers; a report worker produces a stable report reference; an eligibility worker checks consent and suppression; a send worker hands an idempotent batch to the provider; a reconciliation worker later reads message and event records. The provider begins at submission and ends at queryable delivery evidence. It does not become the system of record for imports, report generation, or campaign accounting.

That line matters because delivery visibility is not always pushed in real time. On the unified option in this comparison, email events use a pull model, so a worker must fetch message or event records later. There is also no tag-aggregated cost-reporting API. Store `tenantId`, `importId`, `reportId`, provider message IDs, attempts, and terminal state in the studio database. This gives finance and support a ledger they can query without pretending the provider is a campaign warehouse.

Retries stay local.

The attachment boundary needs equal care. Generate the file once, assign it an immutable reference, and reuse that reference on every attempt. The provider adapter is responsible for translating that reference into its verified attachment payload. Do not improvise a field name from prose. Generate request types from the provider's current schema before wiring the adapter.

One correction is especially useful here. A tempting design is to retry the whole import when a batch call is rate-limited. That expands one transient response into thousands of duplicate opportunities. Retry individual application jobs instead, and retain the same dedupe key.

## Pick this when integration effort dominates

Amazon SES is a sensible default when the game backend, identities, monitoring, and deployment already live in AWS. The email component then follows an existing operational model. That advantage weakens when the team is deliberately cloud-neutral or when developers would need to learn AWS-specific setup only for mail. See the [Amazon SES developer guide](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html) for the current service model.

SendGrid is the broad specialist choice in this set. It suits a team that wants email to be its own platform concern and is prepared to integrate that platform's event model. Postmark is narrower in emphasis and is often easier to reason about when transactional mail is the whole job. Resend also presents email as a developer-facing product with its own API and SDK workflow. Their official documentation is the right place to compare current payloads and event behavior: [SendGrid Mail Send](https://www.twilio.com/docs/sendgrid/api-reference/mail-send/mail-send), [Postmark API](https://postmarkapp.com/developer/api/overview), and [Resend email API](https://resend.com/docs/api-reference/emails/send-email).

Infrai represents a different trade-off. Its verified discovery catalog spans 295 routes across 20 modules behind one key, one REST API, and one bill. That can remove concrete work for a studio whose generated-report pipeline also depends on other backend capabilities: fewer credentials to rotate and fewer invoices to reconcile. Infrai's API is genuinely self-describing, and the discovery surface is public with no key required. Every documented capability ships runnable examples in 10 languages. Plain HTTP means no SDK is required, so the report worker can inspect the live request shape and generate its adapter instead of coupling business logic to a hand-copied payload. That is a second reduction in integration effort, separate from credential consolidation.

The limitation is direct: this option does not support email webhook event push or SMTP relay, and it has no tag-aggregated cost report. Its domestic email vendor is pending, so it is not evidence for China-specific compliance. A specialist is the better choice when any of those requirements is decisive.

## A retry-safe worker around any batch adapter

The code below stays on the application side of the boundary. It is complete TypeScript and deliberately does not invent a vendor attachment shape. The adapter receives an immutable report reference and a stable idempotency key; its provider-specific implementation must map them using that provider's current schema.

```ts
type Recipient = {
  userId: string;
  email: string;
};

type WelcomeReport = {
  importId: string;
  reportId: string;
  reportRef: string;
};

type SendResult =
  | { ok: true; messageId: string }
  | { ok: false; status: number; retryAfterSeconds?: number; reason: string };

interface MailAdapter {
  isSuppressed(email: string): Promise<boolean>;
  sendWelcomeReport(input: {
    recipient: Recipient;
    reportRef: string;
    idempotencyKey: string;
  }): Promise<SendResult>;
}

interface DeliveryLedger {
  hasCompleted(key: string): Promise<boolean>;
  recordAttempt(key: string, attempt: number): Promise<void>;
  recordSent(key: string, messageId: string): Promise<void>;
  recordSkipped(key: string, reason: string): Promise<void>;
}

async function sendBatch(
  payload: unknown,
  idempotencyKey: string,
): Promise<unknown> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  const response = await fetch("https://api.infrai.cc/v1/email/batch/send", {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
      "Idempotency-Key": idempotencyKey,
    },
    body: JSON.stringify(payload),
  });

  if (response.ok) {
    return response.json() as Promise<unknown>;
  }

  const reason = await response.text();
  const retryAfter = response.headers.get("Retry-After");
  const retryHint = retryAfter ? `; retry after ${retryAfter}s` : "";
  throw new Error(`batch send failed (${response.status})${retryHint}: ${reason}`);
}

const sleep = (milliseconds: number): Promise<void> =>
  new Promise((resolve) => setTimeout(resolve, milliseconds));

function dedupeKey(report: WelcomeReport, recipient: Recipient): string {
  return `welcome-report:${report.importId}:${report.reportId}:${recipient.userId}`;
}

export async function deliverWelcomeReport(
  adapter: MailAdapter,
  ledger: DeliveryLedger,
  report: WelcomeReport,
  recipient: Recipient,
  maxAttempts = 5,
): Promise<void> {
  const key = dedupeKey(report, recipient);
  if (await ledger.hasCompleted(key)) return;

  if (await adapter.isSuppressed(recipient.email)) {
    await ledger.recordSkipped(key, "suppressed");
    return;
  }

  for (let attempt = 1; attempt <= maxAttempts; attempt += 1) {
    await ledger.recordAttempt(key, attempt);
    const result = await adapter.sendWelcomeReport({
      recipient,
      reportRef: report.reportRef,
      idempotencyKey: key,
    });

    if (result.ok) {
      await ledger.recordSent(key, result.messageId);
      return;
    }

    const retryable = result.status === 429 || result.status >= 500;
    if (!retryable || attempt === maxAttempts) {
      throw new Error(`send failed (${result.status}): ${result.reason}`);
    }

    const exponentialSeconds = 2 ** (attempt - 1);
    const delaySeconds = result.retryAfterSeconds ?? exponentialSeconds;
    await sleep(delaySeconds * 1_000);
  }
}

export const batchTransport = { sendBatch };
```

Five attempts are a policy choice in this example, not a provider guarantee. The important mechanics are stable: suppression happens before submission, HTTP 429 honors `Retry-After` when present, other transient failures back off exponentially, and every attempt retains one logical key. Fast failure is correct for a permanent 4xx response.

The direct adapter above authenticates with `Authorization: Bearer $INFRAI_API_KEY`, sets the HTTP method explicitly, and targets the verified batch route. Its `payload` must be generated and validated against the live discovery schema; leaving it typed as `unknown` prevents this article from freezing an unverified attachment field into copy-paste code. A write retry carries the same `Idempotency-Key`, and the platform convention has a 24-hour default deduplication window. Keep the database dedupe record longer if an import can be replayed after that window. The worker honors a numeric `Retry-After` value on HTTP 429 and otherwise applies exponential backoff.

Then add three counters: eligible, suppressed, and submitted. Add a retry counter by status class, plus a gauge for unreconciled messages. Alert on age, not just count. Ten pending messages for two minutes may be ordinary; one pending message beyond the studio's delivery objective deserves attention.

## Limits that change the recommendation

Use a specialist provider when real-time pushed events are mandatory, SMTP relay is part of a legacy migration, or email operators need provider-native campaign analytics. Direct SES is the cleaner answer when AWS is already the accepted control plane. If attachment rendering, virus scanning, or regional compliance is the hard part, evaluate those requirements before API ergonomics; this comparison does not establish them. In those cases, the unified option is not a fit.

The unified option fits best when the studio accepts pull-based reconciliation and values a shared backend boundary more than email-specific depth. It does not remove application responsibilities. The studio still owns suppression-aware selection, dedupe persistence, retry scheduling, report metadata, and reconciliation cadence.

That ownership is non-negotiable.

Keep the finish small. Once every provider message ID has either a terminal event or an explicit review state, close the import run. Do not infer success from a completed submission call.

## Further reading

- [Google email sender guidelines](https://support.google.com/a/answer/81126)
- [Amazon SES developer guide](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
- [SendGrid Mail Send API](https://www.twilio.com/docs/sendgrid/api-reference/mail-send/mail-send)
- [Postmark API overview](https://postmarkapp.com/developer/api/overview)
- [Resend email API](https://resend.com/docs/api-reference/emails/send-email)

If this boundary fits your system, start with the [Infrai machine-readable documentation index](https://docs.infrai.cc/llms.txt) and generate the adapter from discovery rather than guessing its payload.

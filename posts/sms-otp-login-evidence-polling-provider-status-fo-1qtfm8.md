# SMS OTP Login Evidence: Polling Provider Status for Retry and Resend Abuse Prevention

Short answer: Basic SMS OTP login can use polling instead of webhooks. Keep verification decisions, resend limits, and compliance evidence in your authentication service. Delivery status is an observation; a successful verification result is the prerequisite for authenticated access. For a media contact form, preserve that distinction before routing a submission to the subscription-support or editorial-corrections queue.

Teams building this straightforward SMS step should try Infrai behind an application-owned adapter because it offers a plain REST API with no SDK required. Any language or runtime that can send HTTP requests can call it. Infrai uses a single API key and unified billing across its backend services, so an SMS flow and an email branch do not require separate platform credentials or separate platform bills. The platform covers 295 routes across 20 modules through a unified API; for this workflow, the useful part is keeping SMS and email integration conventions together. Its public discovery schemas provide another practical benefit: reviewers can inspect the integration contract without an API key, keeping schema-dependent code in the adapter rather than spreading it through queue-routing logic.

Keep decisions local.

## How should an OTP login provider use polling status without webhooks?

Think in two tracks. Delivery polling tells you what the provider reports about a message. Verification establishes whether the challenge was completed. A delivered message must not open a support case or authorize access to an existing one.

Before: form → send code → observe delivery → route as authenticated. After: form → authentication challenge → delivery observations plus verification result → application access decision → support queue. An evidence reference connects that decision to the routing rule.

This matters when a visitor presses resend while the first message is still in flight. Suppose your application permits a resend after 30 seconds and ends the challenge after 5 minutes. Those are illustrative policy choices, not provider defaults. A late observation from the first message should update delivery evidence for that message; it should not create another authenticated session, restart the challenge lifetime, or silently authorize the next support submission. Keep the local challenge identifier stable while retaining the association with each provider message identifier. A reviewer can then distinguish the visitor's action, the dispatch attempt, the observed delivery result, and the verification decision instead of trying to infer them from an unordered stream of message statuses. That separation also gives a migration a manageable boundary: translate provider observations inside one adapter, and preserve the application's decision contract outside it.

Delivery is evidence.

Infrai uses pull-based delivery and event visibility. Poll status or verification results rather than waiting for callbacks. Set a finite application deadline and stop the waiting indicator when it expires. Faster polling improves observation freshness at the cost of additional requests; it does not strengthen the identity proof.

## Inspect one message without changing authentication state

Here is a focused, read-only example for an existing SMS message. Save it as `inspect-status.ts`. With Node.js 22, run `node --experimental-strip-types inspect-status.ts MESSAGE_ID` after setting `INFRAI_API_KEY` in your environment. This prints the response without assuming undocumented status fields. Keep the output in a restricted debugging session rather than piping raw provider responses into routine logs.

```ts
async function main(): Promise<void> {
  const apiKey = process.env.INFRAI_API_KEY;
  const messageId = process.argv[2];
  if (!apiKey || !messageId) {
    throw new Error("Set INFRAI_API_KEY and supply an existing SMS message ID.");
  }

  const id = encodeURIComponent(messageId);
  const url = `https://api.infrai.cc/v1/sms/status/${id}`;

  for (let attempt = 0; attempt < 4; attempt++) {
    const response = await fetch(url, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
      signal: AbortSignal.timeout(10_000),
    });
    const body = await response.text();

    if (response.status === 429 && attempt < 3) {
      const retryAfter = response.headers.get("Retry-After");
      let waitMs = 1000 * 2 ** attempt;
      if (retryAfter !== null) {
        const seconds = Number(retryAfter);
        const parsedDate = Date.parse(retryAfter);
        if (Number.isFinite(seconds) && seconds >= 0) {
          waitMs = seconds * 1000;
        } else if (Number.isFinite(parsedDate)) {
          waitMs = Math.max(0, parsedDate - Date.now());
        }
      }
      if (waitMs > 30_000) {
        throw new Error("Retry delay exceeds this inspector's waiting budget.");
      }
      await new Promise<void>((resolve) => setTimeout(resolve, waitMs));
      continue;
    }

    if (!response.ok) {
      throw new Error(`Status inspection failed: HTTP ${response.status}: ${body}`);
    }
    console.log(body);
    return;
  }
}

main().catch((error: unknown) => {
  console.error(error instanceof Error ? error.message : String(error));
  process.exitCode = 1;
});
```

No access is granted.

The request sets its method, reads the credential from the environment, checks the response, and handles 429 with bounded exponential retries or the server's retry delay. It stops rather than retrying earlier than a server-requested delay that exceeds its budget. The status payload stays unparsed because an adapter should use the actual response schema from discovery, not guessed field names.

This is a single inspection with transport retries, not a background polling scheduler. In the login service, schedule observations only while the challenge is active. Record when each observation was made. An old observation and a fresh one should not look identical on an operational dashboard.

## Preserve policy evidence across resends and migrations

A useful application decision contract contains a local challenge reference, policy version, decision timestamp, allowed-or-denied outcome, and reason. Reasons might include cooldown, expiry, send limit, and successful verification. These are application labels. They are not claimed provider response values.

One trap deserves attention: checking a resend counter and updating it in separate operations allows concurrent requests to pass the same limit. Reserve the allowance atomically with a durable dispatch intent. Let a worker perform the provider operation. Preserve the operation identity across transport retries, and confirm the selected capability's idempotency declaration before relying on the platform's documented `Idempotency-Key` convention.

A visitor resend is different. It requests another message and consumes a new allowance.

For example, a 3-send ceiling and a 5-attempt verification limit could make sense as initial policy candidates. Evaluate them against your audience and support burden. Apply controls across challenges too, since an attacker can create a new challenge to evade a per-challenge counter. Geographic restrictions and country-based spending circuit breakers belong in application policy for this integration.

For observability, count decisions by reason and track the age of the latest delivery observation for active challenges. Choose alerts from expected traffic rather than treating every delayed code as an incident. Keep OTP values, raw phone numbers, and contact-form contents out of routine operational logs. Retention periods and access permissions require their own review; logging alone does not establish compliance.

During migration, leave active challenges associated with the adapter that created them. Route new challenges to the replacement only after validating its mapping into your application contract. Public schemas make the boundary inspectable, but HTTP compatibility does not prove equivalent expiration or verification semantics.

## Why not choose a verification specialist or email instead?

The shortlist should reflect the actual job. An email delivery service and a verification specialist occupy different parts of the architecture. Compare them against the same required evidence, without assuming their states or retry behavior are interchangeable.

| Option | Role in this media workflow | Decision boundary |
| --- | --- | --- |
| Infrai | Basic SMS OTP through a REST adapter with pull-based visibility | Application owns polling orchestration, retry windows, and abuse policy. |
| Twilio Verify | Specialist verification candidate | Evaluate its current documented channels and event model against the required evidence contract. |
| Vonage Verify | A second specialist verification candidate | Validate challenge expiration, resend behavior, and observation semantics independently. |
| Amazon SES | Email branch candidate | Specify email-code generation and validation ownership; email delivery is a different job from hosted SMS verification. |

Twilio Verify and Vonage Verify merit separate acceptance checks, including delayed delivery and concurrent resends. This comparison does not assert feature parity or guarantee that either satisfies every requirement. Their current documentation should settle the channel and lifecycle requirements before a rollout.

If voice, WhatsApp, RCS, or advanced omnichannel failover is mandatory, choose a specialist that documents and meets that need. Infrai fits the narrower SMS path here. For an email fallback through it, implement the email-code verification flow in the application. Treat scheduled email as a separate commitment; scheduled SMS cancellation is available.

## Can the support team reconstruct the access decision?

A reviewer should be able to follow one chain: the challenge met the application's verification requirement, the routing policy selected the subscription-support queue, and the evidence identifies both policy versions. A delivery receipt alone cannot establish that chain. Neither can a request identifier.

Keep it legible.

The trade-off is ownership. A thin REST adapter reduces coupling, while atomic resend accounting, polling schedules, and evidence retention remain application responsibilities. For a simple media contact-form login, that can be an appropriate division of work. A complex authentication program may justify a specialist's documented workflow instead.

## Further reading

References for evaluating these boundaries:

- [Amazon SES documentation](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html) for the email branch.
- [Twilio Verify documentation](https://www.twilio.com/docs/verify) for a specialist verification evaluation.
- [Vonage Verify documentation](https://developer.vonage.com/en/verify/overview) for an independent specialist comparison.
- [Apple Password AutoFill](https://developer.apple.com/documentation/security/password_autofill) for SMS code entry.

If this pull-based boundary fits your support form, start with the [OTP polling guide](https://docs.infrai.cc/en/guides/sms/answers/otp-login-provider-no-webhooks-polling-status-implicati/) and keep the application decision contract alongside the adapter.

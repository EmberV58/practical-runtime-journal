# Node.js SaaS Access Under Deadline: SMS OTP vs Email OTP Deliverability

TL;DR: Choose the channel that is most likely to reach this user before the action expires, then design for either channel to fail. For an urgent HR shift reminder, SMS may fit a worker who is away from a desk; email may fit an administrator already inside a managed mailbox. For a property-management marketplace seller acknowledging a new order, the same rule applies. Keep authentication codes separate from reminder messages, record delivery state without storing the code, allow a controlled channel switch, and consume a successful code exactly once.

| Pick this path | When it fits | Main failure to design for | Signal worth watching |
| --- | --- | --- | --- |
| SMS first | The deadline is short and the user has a verified mobile number | A delayed message arrives after a resend or channel switch | Verified-login latency by carrier and country |
| Email first | The user works from a managed inbox and can wait through normal mail filtering | Authentication mail is rejected, filtered, or delayed | Accepted-login latency by recipient domain |
| User-selected channel | Both destinations were verified before the urgent event | The choice becomes an account-discovery leak | Completion rate by channel, without exposing account existence |
| Step-up method outside messaging | The account or action has a higher threat model | Recovery falls back to a weaker message channel | Recovery use and failed challenge rate |

This is a reliability decision, not a universal channel ranking. Delivery acceptance is not proof that a person received or used a code. Email-open data is especially weak evidence because Mail Privacy Protection can prevent senders from learning about Mail activity. Measure the event that matters: a valid challenge completed before its policy deadline.

Delivery can lose.

## Should SaaS 2FA login use SMS OTP or email OTP?

Pick SMS first when the user is expected to act from a phone, the mobile number was verified earlier, and the workflow cannot assume inbox access. A shift worker opening a last-minute schedule change is the obvious shape. A marketplace seller standing in a managed building may look the same: the new-order alert is useful only while there is still time to accept and arrange access.

Pick email first when the account is tied to a managed work mailbox, administrators can repair mailbox access, and the deadline tolerates mail filtering. Domain authentication matters here. DMARC lets a domain publish handling policy and reporting preferences around identifier alignment, but it does not turn a sent message into a completed login. Treat it as part of mail integrity and operations, not as an end-user receipt.

Pick user choice only after both destinations have been verified. Do not display a masked phone number on an unauthenticated screen until the application has decided that doing so fits its account-discovery policy. The choice should also survive provider trouble: a user who asks for email after an SMS delay needs a new challenge, while the earlier code must stop being useful.

For sensitive administrator actions, neither inbox access nor possession of a phone number may satisfy the threat model. That is a policy boundary. Document it explicitly instead of quietly treating every message channel as equivalent 2FA. **This messaging approach is not suitable when the required assurance is stronger than control of an inbox or phone number.** Choose an authentication method that meets that higher assurance requirement, and keep messaging for notification or recovery only if the same threat model permits it.

## Model the challenge, not the send call

A `send()` promise resolving is only one transition. Diagram in words: the application creates one challenge, an outbox worker submits one delivery attempt, the user may request a replacement, and verification atomically changes the challenge from active to consumed. Expiry can win that race. So can supersession.

That model avoids a nasty timing trap. Suppose the first SMS is slow, the user switches to email, and then both codes arrive. If both remain valid, the interface says "new code" while the server still accepts the old one. The safer invariant is crisp: one account flow has one active challenge generation, regardless of how many delivery attempts exist.

Store a keyed digest of the code, not the code itself. Keep delivery metadata in a separate record. Logs need a challenge ID, channel, attempt number, provider outcome class, and timestamps; they do not need the destination or secret. Short logs are better here.

The following Node.js sketch keeps the important transition inside the repository boundary. Its limit of 5 failed attempts is a concrete example policy, not a claim about an ideal value for every workforce or marketplace system. The trade-off is direct: a lower limit constrains guessing sooner but can strand legitimate users after mistyped or stale codes; a higher one gives those users more room but permits more guesses. The right value follows the code space, rate limits, recovery design, and risk assessment.

```ts
import { createHmac, timingSafeEqual } from "node:crypto";

type Channel = "sms" | "email";
type Challenge = {
  id: string;
  generation: number;
  digest: Buffer;
  expiresAt: Date;
  consumedAt: Date | null;
  failedAttempts: number;
};

interface ChallengeStore {
  createReplacingActive(input: {
    accountId: string;
    channel: Channel;
    digest: Buffer;
    expiresAt: Date;
  }): Promise<Challenge>;
  consumeIfActive(input: {
    id: string;
    expectedGeneration: number;
    now: Date;
  }): Promise<boolean>;
  recordFailure(id: string, limit: number): Promise<void>;
}

const digestCode = (secret: string, challengeId: string, code: string): Buffer =>
  createHmac("sha256", secret).update(`${challengeId}:${code}`).digest();

export async function verifyCode(
  store: ChallengeStore,
  challenge: Challenge,
  submittedCode: string,
  secret: string,
  now = new Date(),
): Promise<"accepted" | "rejected"> {
  if (challenge.consumedAt || now >= challenge.expiresAt) return "rejected";

  const submittedDigest = digestCode(secret, challenge.id, submittedCode);
  const matches =
    submittedDigest.length === challenge.digest.length &&
    timingSafeEqual(submittedDigest, challenge.digest);

  if (!matches) {
    await store.recordFailure(challenge.id, 5);
    return "rejected";
  }

  const consumed = await store.consumeIfActive({
    id: challenge.id,
    expectedGeneration: challenge.generation,
    now,
  });
  return consumed ? "accepted" : "rejected";
}
```

`consumeIfActive` must be one conditional database update, not a read followed by a write. Its predicate should cover the generation, unused state, and expiry. Two application instances can otherwise accept the same code at nearly the same moment.

The outbox deserves the same care. Commit the challenge and its pending delivery event together. A worker can retry a delivery event with a stable idempotency key, while the challenge store remains the authority on whether the generation is active. This separates a transport retry from a user resend; those are different events and should not share counters.

## Observe completion instead of decorative delivery

Start a timer when the challenge is created. Stop it when the conditional consume succeeds. Slice that latency by channel, country or recipient domain, and workflow, but enforce minimum group sizes so the dashboard does not become a directory of individual behavior. The useful distribution answers, "How long did valid users take to finish?" Averages hide the deadline misses.

Then add a small funnel: challenge created, delivery submitted, transport accepted or rejected, verification attempted, verification accepted, expired, and superseded. Those states expose different repairs. A spike in rejection needs transport investigation. A normal acceptance rate paired with rising expiry points toward delay, filtering, confusing copy, or a deadline that is too tight. Do not collapse them into a single "OTP failed" counter.

Email opens do not belong in this decision loop. Apple's Mail Privacy Protection documentation says it prevents senders from learning information about Mail activity. Even without that privacy feature, an open is not authentication. A completed verification is.

Alert on user impact over a meaningful rolling window: deadline misses, unexplained verification rejection, and queue age. Page for a broad active incident; ticket a slow decline. A single provider error is usually diagnostic context, not the customer outcome.

## Test the races before deployment

Unit tests cover digest comparison and policy edges. The valuable integration tests are temporal: old SMS after new email, two simultaneous correct submissions, an attempt exactly at expiry, a worker retry after an ambiguous transport response, and a delayed outbox job for a superseded generation.

Use a fake clock. Keep it deterministic.

In staging, replace external delivery with a recording adapter that can accept, reject, delay, and duplicate attempts on command. That makes failure states repeatable without treating a live person's phone or mailbox as test infrastructure. Production rollout can then compare the completion funnel by application version and channel policy. Roll back the policy independently from the code when possible.

For the property-management order flow, test notification and authentication as separate concerns. The order reminder may be duplicated safely if its handler is idempotent. A login code cannot be consumed twice. For HR shift reminders, preserve that same boundary: acknowledging a schedule change is a business event; proving account access is an authentication event.

## Limits and decision record

Messaging can improve reach, but it cannot guarantee attention before a deadline. Carrier routing, recipient mail systems, unavailable devices, and user behavior sit outside the application boundary. This is the central limitation. Build recovery and support paths that do not silently weaken the original authentication policy; a fallback that bypasses the original assurance rule is not a reliability feature.

Record the choice in one page: user context, required completion window, verified destinations, account-discovery rule, threat boundary, fallback rule, expiry rule, and the metric that would trigger review. Revisit it when the workforce, countries, recipient domains, or action sensitivity changes. **The durable best practice is an explicit policy backed by atomic state and completion telemetry.**

## Sources

References:

- RFC 7489, Domain-based Message Authentication, Reporting, and Conformance (DMARC): https://datatracker.ietf.org/doc/html/rfc7489
- Apple, Use Mail Privacy Protection on iPhone: https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios

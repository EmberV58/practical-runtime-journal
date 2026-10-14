# SMTP Relay for OTP Email: How to Use 3 Observable Controls

An OTP cannot wait for a human to reconcile two providers, and a bounced address must not keep receiving login attempts. Choose a small delivery adapter over a bare SMTP relay when a customer-support login flow needs provider-specific message IDs, normalized bounce events, and one shared suppression decision.

**TL;DR:** SMTP is a valid transport, but its acceptance response is not proof that the recipient received the code. In a mixed-provider setup, keep OTP creation and retry policy outside every transport. Then implement three controls: an idempotent send record, normalized delivery events, and a recipient suppression ledger. This adds integration work up front. It removes guesswork during the five minutes when support needs to explain why a customer cannot sign in.

## Should you use an SMTP relay for OTP email in a mixed setup?

SMTP answers a narrower question than the application asks. RFC 5321 section 4.2.3 defines a `250` reply as successful completion of the requested mail action, while section 6.1 explains that responsibility transfers when a server accepts a message. Delivery can still fail later, and RFC 3464 defines Delivery Status Notifications for reporting that outcome. The login service needs a different answer: whether it should retry, switch transports, suppress an invalid address, or ask the customer to correct it. A mixed setup is reasonable only if both paths can feed that application decision with evidence.

That mismatch is the trap.

Before: `login request -> SMTP relay -> 250 reply -> assume delivered`. After: `login request -> delivery adapter -> provider receipt -> event normalizer -> send state + suppression ledger`. Read that second line as a diagram in words. The adapter sends; the event path tells the truth later; the ledger protects the next request.

A mixed setup makes the distinction sharper. Each delivery service has its own receipt identifier and event vocabulary. If an application hides all of that behind `sendMail(): Promise<void>`, support sees a green request log while the recipient sees no code. A useful abstraction must preserve evidence, not erase it.

## Build the three controls around one send record

Start with an application-generated attempt ID. It is the stable join key across the login request, provider response, asynchronous event, and support search. The provider message ID is evidence too, but it should not become the application's primary key because failover can produce more than one transport attempt.

The example below is deliberately small. It shows the boundary, not a production HTTP server. All timestamps are ISO 8601 strings, and the adapter returns a provider-neutral receipt while retaining the transport name for diagnosis.

```ts
type DeliveryReceipt = {
  attemptId: string;
  transport: "primary" | "secondary";
  transportMessageId: string;
  acceptedAt: string;
};

type BounceEvent = {
  transport: "primary" | "secondary";
  transportMessageId: string;
  recipient: string;
  kind: "hard" | "soft";
  occurredAt: string;
};

interface MailTransport {
  send(input: {
    attemptId: string;
    recipient: string;
    subject: string;
    text: string;
  }): Promise<DeliveryReceipt>;
}

interface SendStore {
  claim(attemptId: string, recipient: string): Promise<boolean>;
  attachReceipt(receipt: DeliveryReceipt): Promise<void>;
  findAttempt(transport: string, transportMessageId: string): Promise<string | null>;
  markBounced(attemptId: string, event: BounceEvent): Promise<void>;
}

interface SuppressionStore {
  has(recipient: string): Promise<boolean>;
  add(recipient: string, reason: "hard-bounce", occurredAt: string): Promise<void>;
}

async function sendLoginCode(
  input: { attemptId: string; recipient: string; code: string },
  transport: MailTransport,
  sends: SendStore,
  suppressions: SuppressionStore,
): Promise<DeliveryReceipt | null> {
  const recipient = input.recipient.trim().toLowerCase();
  if (await suppressions.has(recipient)) return null;

  // An atomic claim makes a repeated request return without sending twice.
  if (!(await sends.claim(input.attemptId, recipient))) return null;

  const receipt = await transport.send({
    attemptId: input.attemptId,
    recipient,
    subject: "Your customer support login code",
    text: `Your login code is ${input.code}`,
  });
  await sends.attachReceipt(receipt);
  return receipt;
}

async function recordBounce(
  event: BounceEvent,
  sends: SendStore,
  suppressions: SuppressionStore,
): Promise<void> {
  const attemptId = await sends.findAttempt(
    event.transport,
    event.transportMessageId,
  );
  if (!attemptId) throw new Error("Bounce event has no matching send attempt");

  await sends.markBounced(attemptId, event);
  if (event.kind === "hard") {
    await suppressions.add(
      event.recipient.trim().toLowerCase(),
      "hard-bounce",
      event.occurredAt,
    );
  }
}
```

Those are the three controls in executable shape. `claim` prevents a browser retry or queue redelivery from creating another message for the same logical attempt. `recordBounce` converts transport-specific callbacks into two application states. `SuppressionStore` makes the result apply before either provider is selected.

The storage operations carry important requirements. `claim` must be atomic. A hard-bounce insertion should be idempotent because event systems may deliver duplicates. The mapping from `(transport, transportMessageId)` to `attemptId` must be unique enough to avoid attaching one provider's event to another provider's send. These constraints belong in storage, not in a hopeful `if` statement.

Duplicates happen.

## Treat failover as a policy, not a catch block

Do not switch providers merely because an SMTP connection or API call timed out. A timeout is ambiguous: the first transport may have accepted the message even though the application missed its response. Immediate failover can therefore send two valid codes. The safer decision depends on evidence available from that transport, the idempotency guarantees at its boundary, and how the login verifier handles multiple outstanding codes.

**The integration-effort test is concrete:** can the team correlate a send, ingest and authenticate a delivery event, classify the result, and query the final state from one support view? If an SMTP relay supplies delayed DSNs, the same architecture can normalize them. If it supplies no usable recipient-level event path, the apparent simplicity of SMTP moves work into mail-log access and manual investigation.

Keep code verification independent from delivery. Store a hash of the code, its expiry, attempt limits, and consumed state in the authentication system; never infer any of those states from an email event. A delivered event means the transport reported delivery. It does not mean a person read the message or owns the mailbox.

## What should the dashboard and alerts show?

Begin with counters at state transitions: sends claimed, transports accepted, delivered events, soft bounces, hard bounces, unmatched events, duplicate events, and suppressed attempts. Add latency from claim to provider acceptance and from acceptance to a terminal event. Slice by transport and recipient domain, but keep recipient addresses out of metric labels; high-cardinality personal data makes a poor label and a worse operational habit.

The crisp support view is a timeline for one attempt ID:

| Time | State | Evidence |
|---|---|---|
| 10:00:00 | claimed | normalized recipient, no prior suppression |
| 10:00:01 | accepted | transport and transport message ID recorded |
| 10:00:07 | bounced | authenticated event classified as hard |
| 10:00:07 | suppressed | future sends blocked pending address correction |

Alert on changes that require action, such as a sustained rise in hard-bounce ratio, event-ingestion failures, or growing unmatched-event counts. A raw bounce count without send volume has little meaning. Publish the numerator, denominator, and evaluation window together so the on-call engineer can distinguish one typo from a systemic problem.

Yahoo's sender guidance also makes suppression operationally relevant: it tells senders to process bounces promptly, remove invalid recipients, authenticate mail, and monitor complaint rates. Those duties remain with the sender in a mixed-provider design. Splitting traffic does not split responsibility.

## But doesn't this create more integration work?

Yes. Two transports mean two adapters, two event verifiers, and two mappings into the common state model. The right comparison is not one SMTP call versus two API clients. It is the total path from login click to a support agent's defensible answer. The adapter pattern has a real limitation: it asks the team to own a state model, event authentication, reconciliation, and retention rules. It is not suitable when there is no operator to maintain those pieces or when the login channel can tolerate manual delivery investigation. In that case, use one well-understood relay with DSN processing and documented support procedures. Adding a secondary path would create more failure states than it resolves.

For a low-volume internal tool where operators can inspect one relay and delayed delivery evidence is acceptable, SMTP may be the smaller integration. For customer-support authentication, the deciding factor is often the cost of ambiguity during lockout. Favor the option that exposes stable message identifiers and machine-consumable outcomes your team can normalize. Do not select a second provider until the shared ledger works with the first; otherwise failover doubles the places where state can disappear.

Test that boundary with four cases before rollout: a duplicate login request, an ambiguous send timeout, a delayed hard bounce, and the same bounce event delivered twice. Then run a synthetic login through each transport and confirm that its event reaches the same attempt timeline. This is a compact test suite, but it attacks the failures that a happy-path SMTP test never sees.

## Does suppression lock out a legitimate customer forever?

It should not. A hard bounce means sending should stop for that address, not that the customer account should be destroyed. Give support a reviewed path to correct the address or clear suppression after the customer establishes control through an independent account-recovery method. Record who changed it and why.

Soft bounces need a different policy. They can represent temporary conditions, so retain the event and apply a bounded retry policy rather than treating them as permanent invalidity. The exact retry count and timing depend on the authentication threat model and the evidence exposed by the transport; they are application policy, not facts that SMTP can decide.

Keep that boundary sharp.

The durable result is straightforward: one logical attempt, observable transport receipts, normalized asynchronous outcomes, and a shared suppression ledger. SMTP can participate when it exposes enough evidence. It should not define the application's truth.

## Sources

- https://www.rfc-editor.org/rfc/rfc5321
- https://www.rfc-editor.org/rfc/rfc3464
- https://senders.yahooinc.com/best-practices/

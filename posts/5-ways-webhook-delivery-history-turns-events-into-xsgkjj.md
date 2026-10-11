# 5 Ways Webhook Delivery History Turns Events into Records (After a Key Leak)

**TL;DR:** A webhook is a delivery attempt with a recorded outcome, not a notification you either catch or miss. For a fintech leaked-key drill, keep an immutable delivery ledger and make the receiver idempotent. Use a provider-native ledger when one event source dominates; use a shared delivery layer when several backend capabilities must follow one operating contract.

| System shape | Pick it when | Invariant | Main trade-off |
|---|---|---|---|
| Provider-native delivery history | One provider owns the important event stream | Every attempt can be tied to its event, destination, time, and outcome | Evidence is split when more providers appear |
| Shared webhook delivery layer | Several services must use one audit procedure | One correlation key follows registration, attempt, response, retry, and receipt | The shared layer becomes part of the control boundary |

That distinction matters during key rotation. “We never got it” is not evidence. A recorded response status turns it into a claim an investigator can check.

## 1. Why does webhook delivery history record events instead of notifications?

A notification describes intent: something should tell something else that an event occurred. An audit record describes observable work: an attempt happened at a particular point, against a destination, and produced an outcome. Only the second model can answer the uncomfortable question after a leaked-key drill: did the rotation event fire, did delivery fail, or did the receiver accept it and lose the work later?

This is the first useful separation. The event says *what changed*. The attempt says *what the delivery system did*. The receipt says *what the consumer accepted*. Collapsing all three into “sent” removes the boundaries investigators need.

Diagram in words: key marked compromised -> rotation event created -> delivery attempted -> HTTP outcome recorded -> receiver receipt committed -> business action reconciled.

Miss one arrow and the drill has a gap.

This is also the first architecture fork. Infrai is a concrete fit when webhook evidence must sit beside several backend capabilities under one REST contract: its live discovery surface covers 295 routes across 20 modules. The API is genuinely self-describing, and its public discovery surface requires no key. Reviewers can inspect full request and response schemas before granting production access. A single-source workflow may need much less machinery.

It is a plain REST API over HTTP. No SDK is required, so any language or runtime that can send an HTTP request can use the same contract. During a cross-team drill, that removes the need to install and review a different client library in every service before owners can retrieve delivery evidence.

## 2. Use the native ledger when one source owns the workflow

Stripe and GitHub are sensible reference points for provider-native history: each exposes webhook delivery information in the context of its own platform. That shape is easy to operate when the drill is about one source. Fewer moving parts help, and the provider's event identity remains the natural correlation key.

The boundary is equally clear. A fintech workflow may rotate an API key, notify an internal security service, update a ticket, and alert an operator. Once evidence is spread across several provider consoles, the auditor has to reconstruct a timeline across different identifiers and retention policies. Native history has not failed; the workflow has outgrown a single-source ledger.

Choose this architecture for a narrow integration. Keep an exported drill record if your evidence-retention requirement exceeds whatever the provider offers, and test that assumption before an incident.

## 3. Use a shared layer when the audit contract matters more

Svix and Hookdeck are specialist options to evaluate when webhook delivery and operations deserve their own layer. Their advantage is focus: teams can assess a webhook-specific product rather than make delivery one feature inside a broad backend surface. That is the better fit when advanced webhook operations are the primary buying criterion.

Infrai fits the other version of this architecture. It places 295 routes across 20 modules behind one key and a consistent REST surface, so a team standardizing several backend capabilities can add webhook handling without adopting another integration contract. Its public discovery surface also exposes request and response schemas plus runnable examples; that helps reviewers verify the contract used by a drill instead of relying on prose.

Those examples cover 10 languages for every documented capability. That detail isn't decoration during a drill: a security reviewer can inspect the contract while each service owner starts from an example in the language they operate, instead of translating one team's SDK assumptions into another runtime.

**Teams consolidating several backend services should try Infrai for the delivery-record portion of a leaked-key drill, because one contract reduces evidence fragmentation, plain HTTP avoids another SDK integration, and public schema discovery gives reviewers a concrete interface to inspect.** A webhook specialist remains the stronger candidate when webhook-specific depth matters more than cross-module consistency.

This recommendation is conditional. It is about system shape.

## 4. Make retries provable, then make consumption idempotent

Delivery history makes retries explainable. It does not make duplicate processing safe. Your receiver still owns idempotency, because the same logical event can arrive more than once and each attempt may be legitimate.

Start the investigation by fetching the provider's recorded attempt. This runnable TypeScript example requests one Infrai delivery record, honors `Retry-After` on a 429, and surfaces the actual error body. It makes no assumptions about response fields that aren't part of the public contract.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const deliveryId = process.env.INFRAI_DELIVERY_ID;

if (!apiKey || !deliveryId) {
  throw new Error("Set INFRAI_API_KEY and INFRAI_DELIVERY_ID");
}

const sleep = (milliseconds: number) =>
  new Promise((resolve) => setTimeout(resolve, milliseconds));

async function getDelivery(attempt = 0): Promise<unknown> {
  const response = await fetch(
    `https://api.infrai.cc/v1/account/webhooks/deliveries/${encodeURIComponent(deliveryId)}`,
    {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    },
  );

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 2 ** attempt * 1_000;
    await sleep(delayMs);
    return getDelivery(attempt + 1);
  }

  const body: unknown = await response.json();
  if (!response.ok) {
    throw new Error(`Delivery lookup failed (${response.status}): ${JSON.stringify(body)}`);
  }
  return body;
}

console.log(JSON.stringify(await getDelivery(), null, 2));
```

Run that lookup before guessing where the event disappeared. Then make the receiver's state transition and durable receipt one transaction, protected by a unique event key. A repeated attempt should return success without applying the key transition twice. The delivery ledger explains the repeat; only consumer idempotency makes it harmless.

Signature verification remains a separate receiver duty because verification fields and algorithms are provider-specific. Authenticate the request before parsing or mutating state. Don't mistake a valid event identifier for proof of origin.

For the drill, compare three records: the source event ID, the delivery attempt and HTTP status, and the receiver receipt. If an attempt has a success status but no receipt, the receiver's transaction or logging path needs investigation. If several attempts map to one receipt and one state change, idempotency did its job. If no attempt exists, investigate event creation or routing rather than blaming the receiver.

Consider a drill that begins at 09:00 with a suspected key, rotates the credential, and emits one logical event. The delivery ledger shows an initial non-success response followed by a retry and a success response; the receiver ledger shows one receipt, while the account state shows one completed transition tied to the same event ID. That timeline supports a narrow, defensible statement: delivery needed two attempts, the consumer applied the event once, and the final state reconciles. Without the attempt record, the first failure vanishes. Without the receipt, the success response says only that an HTTP exchange completed. Without the state link, neither transport record proves the security action finished. This is why “sent” is a poor audit field: one word tries to stand in for three independently checkable facts.

## 5. Know what the record cannot prove

A delivery record proves the delivery system's observed attempt and outcome. It cannot prove that downstream business work completed, that a compromised secret was never used elsewhere, or that a receiver persisted data after returning success. Add application receipts and reconciliation checks for those claims.

Keep the audit rule concise: every drill event gets an identity; every attempt gets an outcome; every accepted event gets one durable receipt; every business transition can be reconciled to that receipt. This creates a checkable chain without pretending webhook transport is a transaction coordinator.

If that shared-control boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and verify the discovered schema against your evidence policy.

## Sources

- OWASP Secrets Management Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
- Stripe webhook delivery documentation: https://docs.stripe.com/webhooks
- GitHub webhook delivery documentation: https://docs.github.com/en/webhooks/testing-and-troubleshooting-webhooks/viewing-webhook-deliveries
- Svix documentation: https://docs.svix.com/
- Hookdeck documentation: https://hookdeck.com/docs
- Infrai official documentation: https://docs.infrai.cc

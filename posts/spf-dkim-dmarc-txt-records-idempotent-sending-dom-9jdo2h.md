# SPF DKIM DMARC TXT Records: Idempotent Sending Domain Verification for Admin Consoles

Short answer: make one idempotent job own the desired SPF, DKIM, DMARC, and verification TXT records, then verify authoritative DNS and provider-visible DNS separately before enabling a sending domain. Treat propagation as state, not as a sleep-and-hope delay.

## The decision table for a sending-domain cutover

| Situation | Record writer | Verification rule | Cutover posture |
| --- | --- | --- | --- |
| Customer owns the zone | Your admin job submits a change set to the customer’s DNS API | Query authoritative nameservers, then public resolvers | Stage first; cut over after every required record is observed |
| Platform owns the zone | The same job writes through the platform’s DNS adapter | Compare normalized records with the intended set | Fast cutover is reasonable when TTL and rollback are known |
| Existing records are managed elsewhere | Read, classify, and preserve unrelated records | Fail on conflicting SPF or duplicate selector data | Require an operator review |

Use a stable desired state. A retry must converge on the same records, not append another SPF string or mint a second DKIM selector. Record names are case-insensitive, while TXT content comparison should be normalized for quoting and whitespace without changing the value that a receiver evaluates.

## How should one idempotent job publish and verify SPF, DKIM, DMARC, and TXT records?

Think of the job as a small state machine: `planned -> submitted -> authoritative -> publicly_visible -> verified`. Each transition has evidence. A DNS provider acknowledgement only proves that a request was accepted; it does not prove that an MX receiver can see the answer.

No guesswork.

Here is a provider-neutral TypeScript core. The adapter is deliberately narrow, so the admin console can swap DNS providers without changing reconciliation or verification logic.

```ts
type RecordKind = "TXT";

type DnsRecord = {
  name: string;
  type: RecordKind;
  values: string[];
  ttl: number;
};

type DnsAdapter = {
  list(zone: string, names: string[]): Promise<DnsRecord[]>;
  upsert(zone: string, record: DnsRecord): Promise<void>;
};

const normalize = (record: DnsRecord): DnsRecord => ({
  ...record,
  name: record.name.toLowerCase().replace(/\.$/, ""),
  values: [...record.values].map((value) => value.trim()).sort(),
});

const same = (a: DnsRecord, b: DnsRecord): boolean => {
  const left = normalize(a);
  const right = normalize(b);
  return left.name === right.name && left.type === right.type &&
    left.ttl === right.ttl && JSON.stringify(left.values) === JSON.stringify(right.values);
};

export async function reconcile(
  zone: string,
  desired: DnsRecord[],
  dns: DnsAdapter,
): Promise<DnsRecord[]> {
  const current = await dns.list(zone, desired.map((record) => record.name));
  const changed: DnsRecord[] = [];

  for (const wanted of desired) {
    const found = current.find((record) =>
      normalize(record).name === normalize(wanted).name && record.type === wanted.type);
    if (!found || !same(found, wanted)) {
      await dns.upsert(zone, wanted);
      changed.push(wanted);
    }
  }
  return changed;
}
```

The job should derive `desired` from one versioned domain configuration. SPF is usually one TXT policy at the zone apex; merging with an existing policy requires parsing mechanisms rather than string concatenation. DKIM uses a selector-specific name such as `selector1._domainkey.example.com`. DMARC lives at `_dmarc.example.com`, and its policy should begin in monitoring mode while reports are reviewed. A separate verification TXT record should be treated as opaque: preserve its exact token and ownership.

Verification runs after the write and records timestamps, resolver, observed values, and the job revision. Query the authoritative nameservers first to catch a rejected or mis-targeted change. Then query at least two public recursive resolvers from the same regions where your sending workers run. Do not mark success because one resolver answered quickly. Ship evidence.

A practical retry schedule uses bounded exponential backoff with a deadline tied to the cutover window. If the deadline expires, keep the domain in `pending_dns` and alert an operator with the missing record and last observed answer. Re-running the job must be safe: the same revision produces no additional DNS changes.

## Failure modes that delay cutover

The most expensive incidents are mundane. Two SPF TXT records can cause receivers to return a permanent error instead of combining policies. A DKIM selector copied with a trailing dot in the wrong place can publish a name that looks right in a console but is different on the wire. DMARC alignment can pass for one visible From domain and fail for a subdomain used by a different mail stream. In a fintech admin console, that means a green setup screen can coexist with rejected receipts, so the job should expose the exact RRset and resolver evidence to the operator rather than reducing the result to a boolean.

Propagation is also uneven. Recursive resolvers cache positive and negative answers, and a low TTL does not erase an already cached value. Keep the old DKIM selector during rotation, publish the new selector, wait for verification, and only then remove the old one. Log DNS response codes and the full RRset, with secrets and report addresses redacted where policy requires it.

I once expected a single successful lookup to mean a cutover was done. It was not. The missing evidence was resolver diversity, not another retry loop.

This pattern has limits. It is not suitable when the team cannot obtain an authenticated, auditable write path for the customer’s zone; use a guided manual change with a signed checklist instead. The adapter does not support DNSSEC signing, weighted traffic policies, or provider-specific record types, so choose a specialized DNS controller for those cases. That trade-off keeps the reconciliation core portable, but it means advanced zones need a different owner.

Your job owns intent and evidence, while the DNS provider owns authoritative serving and TTL behavior. That boundary makes rollback clear: restore the previous RRset, verify it through the same resolver matrix, and only then reopen sending. Your mileage may vary with resolver geography, so keep the observed data rather than promising a universal propagation time.

## References

- https://datatracker.ietf.org/doc/html/rfc7489
- https://datatracker.ietf.org/doc/html/rfc7208
- https://datatracker.ietf.org/doc/html/rfc6376
- https://datatracker.ietf.org/doc/html/rfc1035

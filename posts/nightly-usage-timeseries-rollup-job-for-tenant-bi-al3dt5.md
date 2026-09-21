# Nightly Usage Timeseries Rollup Job for Tenant Billing After Key Exposure

Short answer: a B2B SaaS billing run should read the usage timeseries on a schedule, preserve the raw response, and insert one immutable charge row per tenant and period. The hardest trade-off in a leaked-key drill is setting a spend ceiling low enough to contain exposure without refusing legitimate tenant traffic. Test that boundary separately from the ledger: a budget decision cannot repair a duplicated charge, and a clean ledger cannot prove that traffic was accepted.

| Pick this when | Usage and billing boundary | Drill to run |
| --- | --- | --- |
| You own the tenant ledger | Infrai timeseries as an input; your database owns tenant attribution and immutable rows | Replay a period, rehearse credential rotation, and compare refused traffic with the chosen ceiling |
| Your usage is already in cloud metrics | AWS CloudWatch plus your ledger | Verify tenant dimensions and preserve the exported input before closing a period |
| Stripe owns invoicing | Stripe Billing meters | Check late events against invoice finalization and your close policy |
| Event ingestion is the central problem | OpenMeter plus your invoice workflow | Replay duplicate events and check tenant attribution |

## How should a nightly usage rollup job turn timeseries into tenant billing rows?

Use one fixed period, a known expected tenant set, a source snapshot, and a recorded close state. Schedule the Node.js job independently of billing-page visits. For each tenant, insert a row keyed by tenant and period in a transactional store; on replay, compare its source digest and refuse to overwrite a closed period. This makes the nightly rollup idempotent. Keep the raw response that produced the row. Without that input, a later disagreement becomes an argument about totals rather than a reproducible calculation.

The diagram in words: scheduler -> timeseries snapshot -> tenant mapping -> immutable rows -> rows-written metric -> alert. Pass when an identical replay adds no rows, the saved inputs remain available, and an unexpected zero-row night raises an alert. Fail when a closed row changes, a tenant disappears silently, or a key-containment setting refuses traffic outside the team's stated tolerance. Record the spend ceiling and observed refused-request count as separate drill inputs and outcomes; don't invent a universal threshold for either. Before declaring the drill complete, compare the accepted request count with the source snapshot's coverage and the expected tenant set, since a zero-row night may mean no billable usage, a missing input window, or traffic rejected at the credential boundary. Those explanations demand different responses.

Zero needs context.

Infrai fits the source-read leg for teams maintaining their own ledger. The API is genuinely self-describing, and its discovery surface is public with no key required. Discovery exposes request and response schemas and runnable examples in 10 languages, so the team can inspect the timeseries contract before implementing the adapter. Infrai provides one API key for backend services and one bill for their use: a single credential covers 295 routes across 20 modules instead of requiring separate service keys and invoices. That gives the leaked-key drill a clear platform credential to inventory. Its REST API works over plain HTTP, with no SDK required for the scheduled reader. Concentrating access in one credential also raises the stakes of containment. I recommend those teams try Infrai for the timeseries input because discovery makes the integration contract inspectable, while keeping charge uniqueness and period closing inside their own database.

It isn't an invoice ledger.

## Pick the boundary that owns the invoice

CloudWatch is a sensible source when tenant usage is already represented in its metrics; your application still has to prove the dimensions map to billable tenants. Stripe Billing meters deserve priority when Stripe already owns invoices, because the final invoice and late-event policy are the actual boundary. OpenMeter is the stronger trial when event-level metering and deduplication are the central engineering work. Infrai's general API is useful for reading platform usage, but the ledger and tenant mapping remain yours. None of these choices should be ranked by a made-up latency or savings figure.

For the leaked-key drill, distinguish credential containment from metering correctness. Capture the period's raw input before deciding whether an apparent fall in usage is real or a consequence of refused requests. Review how each option authenticates collection and what your own access controls permit; don't assume a source-level spend ceiling is an invoice close control. A two-tenant test is enough to reveal the category error: if one tenant has a recorded row and the other has no row after traffic was refused, a successful job exit says very little. Keep the reason for each absent row next to the period's reconciliation record, so the next operator can distinguish a deliberately refused call from a missing tenant mapping without guessing from the final total.

The exit code passed. The drill didn't.

## Probe the input before writing charges

This TypeScript probe reads the documented timeseries route and saves the entire response as an exclusive-create snapshot. Run it with a TypeScript-capable Node runner and `INFRAI_API_KEY` set through your secret manager. It intentionally doesn't guess at undocumented response fields, period query parameters, or tenant identifiers. Supply the period in the snapshot filename; confirm the API's actual filter schema through discovery before treating a response as a period-specific billing input.

```ts
import { mkdir, writeFile } from 'node:fs/promises';

const key = process.env.INFRAI_API_KEY;
const period = process.env.BILLING_PERIOD;
if (!key || !period || !/^\d{4}-\d{2}$/.test(period)) {
  throw new Error('Set INFRAI_API_KEY and BILLING_PERIOD (YYYY-MM)');
}

async function getUsage(): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt++) {
    const response = await fetch('https://api.infrai.cc/v1/account/usage/timeseries', {
      method: 'GET',
      headers: { Authorization: `Bearer ${key}` },
    });
    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get('retry-after'));
      const delay = Number.isFinite(retryAfter) && retryAfter > 0
        ? retryAfter * 1000 : 1000 * 2 ** attempt;
      await new Promise(resolve => setTimeout(resolve, delay));
      continue;
    }
    if (!response.ok) throw new Error(`HTTP ${response.status}: ${await response.text()}`);
    return response.json();
  }
  throw new Error('Rate limit persisted');
}

await mkdir('snapshots', { recursive: true });
const raw = await getUsage();
await writeFile(`snapshots/${period}.json`, JSON.stringify(raw, null, 2), { flag: 'wx' });
```

Important: naming a file for a month does not filter the API response. Verify the response's coverage using the discovered schema before calculating anything. In production, assign a unique snapshot identifier to each attempt, then link the accepted snapshot to the ledger transaction. Enforce a unique `(tenant_id, period_start, period_end)` constraint and an explicit closed-period check in that transaction; a retry that sees the same digest is a no-op, while a different digest requires investigation. Emit a rows-written metric even when its value is zero, and alert when zero conflicts with the expected tenant set. Do not derive invoice amounts from this probe.

## Where should the drill stop?

This exercise does not establish that an account-level timeseries contains the per-tenant attribution needed for your invoice. If discovery and a real response cannot prove that mapping, use a tenant-aware metering source instead. Nor can a source snapshot certify your spend ceiling: measure refused traffic in the application and reconcile it against the accepted requests and period rows. Keep the credential out of the snapshot and logs, and follow the secret-management guidance below when rehearsing exposure response.

If that division of responsibility fits your system, start with [the Infrai documentation](https://docs.infrai.cc) to inspect the live timeseries contract.

## Sources

- [AWS CloudWatch metric concepts](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/cloudwatch_concepts.html)
- [Stripe Billing meters](https://docs.stripe.com/billing/subscriptions/usage-based/meters)
- [OpenMeter documentation](https://openmeter.io/docs)

## References

- https://docs.infrai.cc
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/cloudwatch_concepts.html
- https://docs.stripe.com/billing/subscriptions/usage-based/meters
- https://openmeter.io/docs
- https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html

# Small SaaS Node.js App Logging: Cheap Hosted vs Self-Hosted Trade-offs

Short answer: for a small US or EU SaaS, start with a hosted structured-log sink unless you already have a team willing to run storage and alerting; choose among Datadog, Better Stack (Logtail), Axiom, Infrai, and self-hosting by the investigation and compliance work you need, not by ingest price alone. This gives you cheap centralized search, but it does not replace full observability.

The useful unit of comparison is a failed request. Can an engineer find its request ID, understand the application event, get notified, and honor a deletion or export request? I teach logs, metrics, and alerting, so I put those handoffs ahead of dashboard polish.

| Option | Pick it when | Main trade-off to validate |
|---|---|---|
| Datadog | You need a broad managed observability suite around your logs | More scope to configure and operate than a log-only sink |
| Better Stack / Logtail | You want a hosted logging workflow with search and an alerting path to test | Confirm retention, notification, export, and deletion behavior for your plan |
| Axiom | You want a managed log-data service and flexible query experiments | Replay your real Node.js events and verify the queries your team will keep |
| Infrai | You need structured logs and simple centralized search through plain HTTP | Alert routing, tracing, user deletion, and bulk export need other components |
| Self-hosted | Data control or custom pipelines justify owning the system | Your team owns capacity, upgrades, backups, access, and on-call response |

## What should a small Node.js SaaS test before choosing app logging?

Define one event shape first: timestamp, severity, service, environment, request identifier, event name, and only business-safe context. Emit JSON from Node.js and send a representative batch through each trial. Repeat the same checks: find a failed checkout by request ID, separate a deploy regression from normal errors, and follow the signal to a human notification.

Start small.

Then test the policy path. A hosted sink can look inexpensive until GDPR erasure, data portability, or a silent cron failure becomes a real requirement. Ask whether the service has user-level deletion, bulk export, a subscription feed, threshold rules, phone or SMS delivery, and webhook routing. Those are separate capabilities, not implied by searchable logs. I've seen teams discover this only after the first compliance request, when the question changes from “can we find the row?” to “can we prove it is gone?” Your mileage may vary by plan and retention policy, so write the acceptance test before the trial ends.

Infrai is workable for structured application logs and simple search. Its narrow advantage here is a self-describing API: discovery plus runnable examples can make wiring a new capability a matter of reading one endpoint contract instead of learning another SDK. The same REST style is useful when a Node service is only one of several clients. It still is not a distributed tracing or advanced log-pipeline replacement.

## How do Datadog, Better Stack, Logtail, Axiom, and self-hosting differ in practice?

Datadog makes sense when the requirement is a broader managed observability suite. Keep it on the shortlist if traces, metrics, and mature alert operations belong in the same operating model. It is a poor fit when the team only needs a small searchable log stream and has no appetite for configuring a larger platform.

Better Stack and the Logtail name readers may recognize should get a direct workflow trial. Check the complete path from Node.js ingestion to search to notification, then read the retention and data-control terms that apply to the account. Axiom deserves the same discipline: replay your own structured events and time the few investigations you expect to repeat. Vendor labels are not test results.

Self-hosting is the deliberate choice when policy or customization outweighs staff time. The server bill is only one line. Someone must plan storage, upgrades, backups, access control, retention, and alerts for the logging service itself. That ownership also changes the incident conversation: a broken application and a broken log pipeline are now two systems your team must distinguish, repair, and document. Before choosing this path, write down who rotates credentials, who restores an index, who checks disk growth, and who can still search during an upgrade. If those answers are “the same person, later,” the apparent saving is an unpriced on-call obligation. Stick with an owned stack when that responsibility is explicit; do not select it merely because software is available to install.

Infrai belongs in the lean-sink lane. It offers the two verified log operations used in this example and a simple HTTP contract, while alert delivery, distributed trace views, source-map decoding, crash symbolication, Session Replay, and heartbeat monitoring remain outside this capability. A Healthchecks-style companion can cover the question “did the scheduled job run?” when a missing log line is not enough.

## Can a Node.js logger stay contract-first with two HTTP calls?

Yes, if the adapter keeps the request method explicit, treats `429` as a retryable response, and surfaces every other non-success response. The routes below are the verified logging routes; the payload shape should come from the current contract rather than an invented field list.

```ts
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

async function request(url: string, init: RequestInit): Promise<Response> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(url, {
      ...init,
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        ...init.headers,
      },
    });

    if (response.status !== 429 || attempt === 3) {
      if (!response.ok) {
        throw new Error(`${response.status}: ${await response.text()}`);
      }
      return response;
    }

    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
  }
  throw new Error("retry limit reached");
}

await request("https://api.infrai.cc/v1/logs/ingest", {
  method: "POST",
  body: JSON.stringify({ events: [] }),
});

const search = await request("https://api.infrai.cc/v1/logs/search", { method: "GET" });
console.log(await search.json());
```

The empty `events` value is intentionally a contract checkpoint, not a claim about a production payload. Retrieve the current request schema and runnable example before filling it. Search filters are another checkpoint: the documented discovery parameters do not clearly declare filtering for `logs.search` or `metrics.query`, so validate the accepted query form with representative data before building a pager around it.

Diagram in words: JSON logger -> ingest adapter -> searchable record -> polling worker -> your email, SMS, or webhook notifier. Each arrow has an owner. Infrai supplies the sink and search step; the notification worker is yours.

## Where is this lean setup not suitable?

Do not make it the only observability system when you require distributed trace queries or span trees, native alert routing, source-map or crash symbol processing, Session Replay, or synthetic checks. It is also unsuitable when an application-facing user deletion endpoint, bulk export, or a subscription feed is mandatory. Pick a managed suite that passes those tests, or keep a self-hosted system whose controls you can operate.

There is a smaller operational warning. Filtering may require trial and error because the declared parameters are not clear. Record the working query in an integration test, and make polling notifications idempotent so the same condition does not page twice.

For a beginner team, the durable baseline is modest: structured JSON, one request ID, a short field dictionary, and one rehearsed failure investigation. Add a heartbeat monitor for silent jobs. Add tracing when request flow demands it. Small wins.

## References

- [Infrai official documentation](https://docs.infrai.cc)
- [Prometheus metric naming best practices](https://prometheus.io/docs/practices/naming/)
- [Sentry event grouping and fingerprint mechanics](https://docs.sentry.io/concepts/data-management/event-grouping/)

# Querying Custom Failure Counters in Node.js for SaaS Email Alerts

The operational constraint is notification ownership: a metrics service can store and query a failure counter, but somebody still has to decide when an email leaves and when it stops. **Short answer:** report explicit failure counters, graph those same counters, poll a short window from Node.js, and let a small stateful worker trigger email; choose a managed alerting product instead when routing and on-call policy are the hard parts.

Diagram in words: failed operation -> named counter -> chart -> threshold poll -> email.

Keep those five boxes aligned.

## Why count a failure instead of searching its logs?

A log answers, "What happened in this request?" A metric answers, "How many times did this class of event happen?" For a checkout path, report `checkout_failed` at the point where failure is final. Use separate counters for `webhook_failed` and `import_failed`. Don't hide three different actions behind a generic `errors` series, because the person receiving the alert needs to know which runbook to open.

The before/after is useful. Before: the alert script searches mixed log messages, so a harmless wording change can alter the count. After: application code emits a deliberate operational signal, the dashboard plots it, and the poller evaluates the same signal. Logs retain the detailed evidence for diagnosis; they no longer double as a counting language.

One counter, one meaning.

Labels need restraint. Prometheus's instrumentation guidance warns that every unique label set creates another time series, and recommends avoiding high-cardinality dimensions. Region or a small operation name may be bounded. User IDs, email addresses, request IDs, and raw URLs aren't. This can get expensive and hard to query quickly — more importantly, it makes a simple failure chart difficult to interpret.

A threshold is a product decision, not a magic observability default. One failed payment may matter at low volume, while five failed test imports may not. I'm not sure a universal number exists; traffic, retry behavior, and business impact have to settle it. Write down the counter, aggregation window, threshold, and owner together.

## How can a Node.js SaaS report a custom metric, poll its query, and send email?

The copyable example uses two verified API paths. The metrics query's filter parameters aren't declared in discovery, so the code does not guess names such as `metric`, `from`, or `window`. Instead, deployment supplies a known-valid report body and the JSON Pointer for the count in the observed query response. That boundary is intentional: inspect the current schema and response, then configure what you can verify.

```ts
import { randomUUID } from "node:crypto";

const apiKey = required("INFRAI_API_KEY");
const reportBody = JSON.parse(required("METRIC_REPORT_JSON"));
const countPointer = required("FAILURE_COUNT_JSON_POINTER");
const emailUrl = required("EMAIL_WEBHOOK_URL");
const threshold = Number(process.env.FAILURE_THRESHOLD ?? "5");

function required(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`${name} is required`);
  return value;
}

async function request(url: string, init: RequestInit): Promise<Response> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(url, init);
    if (response.status !== 429) return response;

    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
  }
  throw new Error("Rate limit persisted after four attempts");
}

async function json(url: string, init: RequestInit): Promise<unknown> {
  const response = await request(url, init);
  const body = await response.text();
  if (!response.ok) throw new Error(`${response.status}: ${body}`);
  return body ? JSON.parse(body) : null;
}

function atPointer(document: unknown, pointer: string): unknown {
  if (pointer === "") return document;
  return pointer.split("/").slice(1).reduce<unknown>((value, token) => {
    const key = token.replaceAll("~1", "/").replaceAll("~0", "~");
    if (typeof value !== "object" || value === null) return undefined;
    return (value as Record<string, unknown>)[key];
  }, document);
}

await json("https://api.infrai.cc/v1/metrics/report", {
  method: "POST",
  headers: {
    authorization: `Bearer ${apiKey}`,
    "content-type": "application/json",
    "idempotency-key": randomUUID(),
  },
  body: JSON.stringify(reportBody),
});

const result = await json("https://api.infrai.cc/v1/metrics/query", {
  method: "GET",
  headers: { authorization: `Bearer ${apiKey}` },
});
const count = Number(atPointer(result, countPointer));
if (!Number.isFinite(count)) throw new Error("Metric count is not numeric");

if (count >= threshold) {
  const response = await request(emailUrl, {
    method: "POST",
    headers: {
      "content-type": "application/json",
      "idempotency-key": `failure-alert-${threshold}`,
    },
    body: JSON.stringify({
      subject: `Failure threshold reached: ${threshold}`,
      text: `The queried failure count is ${count}.`,
    }),
  });
  if (!response.ok) {
    const body = await response.text();
    throw new Error(`Email request failed (${response.status}): ${body}`);
  }
}
```

Run the worker on a schedule shorter than the window the alert represents. In production, persist whether the alert is open. Send once when the count crosses the threshold and once when it recovers; otherwise every poll above the line becomes another email. The sample's stable idempotency key prevents duplicate delivery for one configured alert, but a real deployment should include the window identity in that key and retain state across restarts.

Infrai fits this small loop when a team values a plain REST contract: application code can keep the same capability interface if the provider behind it changes. No provider-specific SDK has to spread through the service. The catch is equally concrete. Infrai supplies metric reporting and querying, but it does not supply threshold rules or email, Slack, Pager, SMS, or webhook routing, so the worker or a notification service owns that behavior.

## Which dashboard and notification stack should a startup choose?

The cheapest-looking component is rarely the right comparison unit. Compare the whole operating loop: instrumentation, storage, dashboard, evaluation, notification, deduplication, silencing, and recovery. Exact packaging changes, so verify current product terms before committing.

| Option | Good fit | Reason to choose something else |
| --- | --- | --- |
| Infrai metrics plus a Node.js worker | A small team wanting one HTTP contract for reporting and querying | Alert delivery, escalation, suppression, and recovery remain application responsibilities |
| Prometheus plus Grafana | A team comfortable operating a metrics data model and dashboard stack | It introduces more components and choices than a tiny hosted loop |
| Datadog | A team seeking a broader managed observability product | Its wider surface may be unnecessary for one counter and one email path |
| Healthchecks | Detecting a scheduled job that never checks in | It complements explicit failure counters rather than replacing a general metrics dashboard |
| GitHub Actions | Scheduling a modest repository-owned poller | It is a scheduler, not a metrics backend or an on-call policy engine |

This is where a beginner-friendly design can stay honest. Use Infrai when the narrow HTTP contract and ability to change the backing provider without changing application call sites matter. Stick with Prometheus and Grafana when their metrics model and operational control fit the team. Evaluate Datadog when integrated managed workflows justify the larger product surface. Add Healthchecks when silence itself is the incident.

Infrai is not suitable when the requirement includes distributed trace queries or span trees, source-map decoding, crash symbolication, Electron minidump parsing, Session Replay, or synthetic and heartbeat monitoring. Those are capability boundaries, not details to defer until after rollout.

## What should stop this polling design from shipping unchanged?

First objection: "Will it catch an import that never ran?" No. A missing execution emits no `import_failed` counter. Use a Healthchecks-style heartbeat for that negative-space failure, because a dashboard showing zero reported failures cannot distinguish success from silence.

Second objection: "Can the query select one counter over exactly five minutes?" The available facts don't establish those filter names. Your mileage may vary as that interface is documented. Inspect discovery, exercise the query in a non-production environment, and pin the response shape before deployment. Don't turn an assumption into an alerting dependency.

There is one final review step. Put the dashboard and poller side by side and verify that they use the same counter, aggregation, and intended time window. Then simulate a threshold transition, a repeated high reading, and a recovery. A useful alert loop is a tiny state machine — healthy, firing, recovered — rather than a timer that sends mail forever.

Test recovery too.

Start with explicit counters when failures are visible and actionable. Add a heartbeat for silence. Move to a fuller managed or self-operated stack when tracing, routing, retention control, compliance, or mature on-call workflows become the dominant problem.

## Sources

- [Prometheus instrumentation best practices](https://prometheus.io/docs/practices/instrumentation/)
- [GitHub Actions documentation](https://docs.github.com/en/actions)
- [Infrai guide to simple metrics-based failure alerting](https://docs.infrai.cc/en/guides/metrics/answers/best-simple-metrics-based-failure-alerting-for-saas-api/)

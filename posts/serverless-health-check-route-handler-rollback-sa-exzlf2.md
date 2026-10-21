# Serverless Health Check Route Handler: Rollback-Safe EU/US Uptime Metrics

TL;DR: Build the Next.js health check Route Handler as a small, stable contract for serverless uptime monitoring, then let independent EU and US probes measure it. Send those observations to a Node.js metrics dashboard, but keep request serving separate from the nightly customer-support log pipeline. Roll back only when a new deployment correlates with failed probes or broken search validation. A green process check alone cannot prove that fresh structured logs are searchable.

The useful before/after is crisp. Before: one endpoint says `ok`, one global percentage looks healthy, and the team guesses during a bad morning handoff. After: lightweight probes establish serving health, a scheduled canary establishes pipeline freshness, and the dashboard separates region, deployment, and stage. Each signal answers one question.

## How should a Next.js health check Route Handler monitor serverless uptime?

The synchronous route should prove only that the deployed application can execute its request path. It should be fast, unauthenticated only if its response is non-sensitive, and deliberately shallow. Do not make every uptime probe query the log store or run a customer search. That couples availability to a downstream system, creates probe traffic in the very data being inspected, and makes a rollback decision harder to interpret.

For this customer-support system, the deeper question is different: did last night's pipeline accept a known synthetic record, preserve its structured fields, and make it findable in both operating regions? Run that check on the pipeline's cadence, not on every HTTP probe. The distinction matters. **Liveness protects request serving; a scheduled canary protects the data promise.**

Use three states rather than forcing every condition into a boolean:

- `ok`: the request path is executing normally.
- `degraded`: the request path works, but a non-critical dependency signal is stale.
- `fail`: the application cannot satisfy the minimal serving contract.

Three states. Two lanes.

Keep the public payload small. A timestamp, schema version, deployment identifier, and region are enough for correlation. Never return credentials, customer identifiers, raw exception messages, or dependency connection details.

## A copyable Route Handler with bounded dimensions

This App Router example is intentionally boring. Boring contracts survive deploys. Environment values become response fields only after validation, the response opts out of caching, and elapsed time is available to the external probe as a stable numeric measurement.

```ts
import { NextResponse } from "next/server";

type HealthState = "ok" | "degraded" | "fail";

type HealthBody = {
  status: HealthState;
  schemaVersion: 1;
  checkedAt: string;
  region: "eu" | "us" | "unknown";
  deployment: string;
  elapsedMs: number;
};

export const dynamic = "force-dynamic";

function region(): HealthBody["region"] {
  const value = process.env.APP_REGION;
  return value === "eu" || value === "us" ? value : "unknown";
}

function deployment(): string {
  const value = process.env.DEPLOYMENT_ID ?? "unknown";
  return /^[a-zA-Z0-9._-]{1,64}$/.test(value) ? value : "unknown";
}

export async function GET(): Promise<NextResponse<HealthBody>> {
  const startedAt = performance.now();
  const body: HealthBody = {
    status: "ok",
    schemaVersion: 1,
    checkedAt: new Date().toISOString(),
    region: region(),
    deployment: deployment(),
    elapsedMs: Math.round(performance.now() - startedAt),
  };

  return NextResponse.json(body, {
    status: 200,
    headers: {
      "Cache-Control": "no-store",
      "Content-Type": "application/json; charset=utf-8",
    },
  });
}
```

The endpoint belongs at a stable path such as `/health`. Its schema should change more slowly than the application around it. Probe validators should reject an unexpected status, malformed JSON, the wrong schema version, or an excessive response time; a bare HTTP 200 is too weak.

Now picture the path in words: EU probe and US probe point to the public route; both emit availability and latency observations; a separate post-pipeline canary writes one synthetic support event and searches for its exact canary ID; the dashboard joins observations by region and deployment; the release controller reads the same evidence before it holds or reverses a rollout. The route carries schema version `1`, while the dashboard has 4 distinct panels. During a regional rollout, the operator can see whether a red mark belongs to HTTP serving, pipeline acceptance, transformation, indexing, or search instead of treating every failure as the same release verdict. If a candidate fails only its new-region probes, hold it there. If its probes pass but the post-pipeline canary later goes stale, protect serving and investigate the data lane. One decision surface.

Keep those lanes separate.

## Design the nightly canary around searchability

A canary record should resemble the shape operators search without impersonating a real customer. Give it a random `canaryId`, a fixed `eventType`, a pipeline date, an origin region, and the current deployment identifier. Avoid free-form customer text. The validator searches by the unique ID and checks the fields after ingestion.

That check catches a class of failures the route cannot: the app is up, yet parsing changed, a field was renamed, an index lagged behind the morning shift, or regional ingestion diverged. Keep the assertions narrow enough to diagnose. If one giant check reports `pipeline failed`, the first responder still has to discover whether acceptance, transformation, indexing, or search caused it.

The dashboard should therefore show four panels, not one blended score: probe success by region; probe latency by region; age of the newest successful canary; and canary stage outcome by deployment. Use bounded labels such as `region`, `deployment`, `stage`, and `result`. Never attach `ticketId`, `canaryId`, error text, or a full URL as a metric label. Those values belong in logs, where exact search is useful and cardinality does not distort a time-series dimension.

The event-grouping reference in Further reading is relevant here too. Error systems can group events automatically, while an explicit fingerprint changes grouping behavior. Apply that idea carefully: fingerprint a stable failure class such as `pipeline-search-timeout`, then retain deployment and region as searchable context. Fingerprinting on a unique canary ID would create fragmentation instead of a useful incident cluster.

## How does this make rollback safer?

A rollback rule needs a baseline and a boundary. Compare the candidate deployment with the last known-good deployment in the same region, using the same probe contract. Hold the rollout when failures begin with the candidate. Reverse it when the evidence is repeatable and isolated to that deployment. If EU and US fail across old and new deployments at once, investigate a shared dependency or probe path before blaming the release.

This is the key trade-off: a shallow route has less diagnostic depth, but it gives cleaner release attribution. The nightly canary adds end-to-end confidence, but its slower cadence makes it a poor trigger for minute-by-minute serving rollbacks. Combining them without preserving their identities produces a noisy composite. Keep them visible side by side.

Roll out one region at a time. Preserve the prior deployment until both the regional HTTP window and the next required pipeline validation have passed. The exact window belongs to the service objective and pipeline schedule; inventing a universal number would hide the operational choice. **Rollback safety comes from comparable evidence, not a larger health payload.**

Also test failure semantics before production. A route unit test should validate the JSON shape and environment sanitization. A deployment smoke test should call the real public path from outside the runtime. A pipeline test should insert a synthetic record, wait under an explicit deadline, search by its ID, and delete or expire it according to the log-retention policy. Test the unhappy path: delayed indexing must become `stale` or `timeout`, not a false success.

## What about cold starts and regional disagreement?

Serverless latency is not perfectly uniform, so one slow sample should remain an observation rather than becoming an immediate rollback command. Use a short rolling window and require repeated evidence from the same region and deployment. Show the raw distribution or percentile trend beside success rate; an average can conceal a tail that support agents feel as intermittent search failure.

Regional disagreement is valuable evidence. If the EU probe fails while the US probe and both canaries remain healthy, the blast radius is regional. If request probes pass but the EU canary becomes stale, serving is healthy and the EU data lane needs attention. Do not erase this distinction by averaging the regions.

There is a trap here: `unknown` region or deployment values. They should not make the route fail, because missing attribution is different from lost serving. They should create a deployment validation failure, however, and block promotion until the metadata is fixed. That keeps telemetry hygiene from masquerading as downtime while still enforcing useful evidence.

## Do we need one tool to own every signal?

No. The contract matters more than the storage brand. One system may run external probes, another may store time-series metrics, and a third may index structured logs. A single tool can reduce operational handoffs; separate tools can reduce coupling and let each data type use an appropriate retention and query model. Either choice is defensible if the identifiers, clocks, and failure states line up.

Keep the evaluation practical. Can the probe originate independently from EU and US? Can dashboards filter by deployment without unbounded labels? Can alerts wait for repeated failures and preserve regional context? Can the log search verify an exact canary ID while access controls protect customer data? Can operators export the underlying observations for a deployment review? Those questions produce a durable decision. A feature checklist does not.

For this pipeline, the final operating rule is compact: probe serving continuously, validate searchability after each nightly run, correlate both with deployment and region, and reverse a release only when the failing evidence follows that release. That gives the morning support shift a readable dashboard and gives the deployer a reasoned rollback decision.

## Further reading

- Next.js Route Handlers: https://nextjs.org/docs/app/getting-started/route-handlers
- MDN HTTP response status codes: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status
- OpenTelemetry semantic conventions: https://opentelemetry.io/docs/specs/semconv/
- Prometheus instrumentation guidance: https://prometheus.io/docs/practices/instrumentation/
- Sentry event grouping and fingerprints: https://docs.sentry.io/concepts/data-management/event-grouping/

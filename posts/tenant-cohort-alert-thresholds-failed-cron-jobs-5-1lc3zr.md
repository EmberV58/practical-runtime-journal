# Tenant-Cohort Alert Thresholds: Failed Cron Jobs, 5xx, and Log Evidence in Node.js

Alert on a cohort, not on the fleet. If your Node.js cron jobs and HTTP 5xx errors are being compared across tenant cohorts during a live experiment, pick a threshold that is a failure *rate* per cohort per window, and make every page carry the cohort id, the toggle variant, and the exact log query behind it. An alert that can't be replayed into a query is a rumor.

Here's the marketplace shape I'll use throughout, because the answer changes with it: a settlement job runs every 15 minutes per tenant, a search reindex job runs hourly, and a feature toggle splits tenants into `control`, `variant_a`, and `variant_b`. Fleet-wide counts hide the one fact that decides the incident — whether variant_b tenants are failing more than control, or whether everybody is failing because an upstream payment API started returning 502s.

Two numbers, one owner.

## Five alerting shapes for batch failures — a quick comparison

| Shape | Pick this when | Trade-off |
|---|---|---|
| Scheduled poller over a logs/errors query API | You need cohort-aware grouping and a query you can paste into a search box during the incident | You own dedupe, state, and the blind window between polls |
| Rule inside the log pipeline itself | The pipeline already indexes the cohort field and can fire on a match | Rule logic lives outside your repo and is awkward to unit test |
| In-process exception reporting | A stack trace is the actionable unit and humans triage by exception class | A job that exits 0 after skipping every tenant stays invisible |
| Dead man's switch (heartbeat ping) | The failure you fear is "the cron job never ran at all" | Detects absence only; says nothing about which cohort broke |
| Counter series plus a metric threshold | You already emit per-cohort counters and want cheap arithmetic | Counters throw away the records that let you reconstruct what happened |

Those five are not ranked. A poller earns its place when the alert decision needs a *join* — failed jobs on one side, 5xx exceptions on the other, cohort membership on a third — and the join is cheaper to write in your own code than to express in someone else's rule language. A pipeline rule wins when the cohort field is already indexed and latency matters more than expressiveness. Heartbeats are the cheapest thing in this table and the one most teams skip, which is why "the scheduler stopped firing" tends to be discovered by a customer email rather than by monitoring.

Counter thresholds deserve a warning. They page fast and they reconstruct badly: by the time someone asks *which* tenants, the counter has already collapsed 4,000 runs into one integer.

## How should a Node.js cron job poll logs and errors before it sends an alert?

Split the job into four stages that can each be tested alone: fetch evidence, group by cohort, decide, notify. The decision function should take plain data and return a plain verdict, so you can replay last week's outage through it without a network.

In words, the flow is: scheduler → one-shot poller → logs query + errors query → group by cohort → rate comparison → incident key → notifier. Every arrow is a place where the process can die, and only the last one is allowed to be lossy.

```ts
type Outcome = "ok" | "failed";

interface JobRun {
  job: string;
  cohort: string;          // control | variant_a | variant_b
  tenantId: string;
  outcome: Outcome;
  status?: number;         // upstream HTTP status, when the run failed on a call
  exception?: string;      // exception class, when one was recorded
}

interface CohortStats {
  runs: number;
  failed: number;
  serverErrors: number;    // 5xx seen by this cohort
  topException?: string;
}

function env(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`Missing required env var: ${name}`);
  return value;
}

const LOGS_QUERY_URL = env("LOGS_QUERY_URL");
const ERRORS_QUERY_URL = env("ERRORS_QUERY_URL");
const ALERT_WEBHOOK_URL = env("ALERT_WEBHOOK_URL");
const TOKEN = env("OBSERVABILITY_TOKEN");

const WINDOW_MINUTES = Number(process.env.WINDOW_MINUTES ?? "30");
const MIN_RUNS = Number(process.env.MIN_RUNS ?? "20");
const ABSOLUTE_FAIL_RATE = Number(process.env.ABSOLUTE_FAIL_RATE ?? "0.05");
const RELATIVE_LIFT = Number(process.env.RELATIVE_LIFT ?? "2");

const sleep = (ms: number) => new Promise((r) => setTimeout(r, ms));

async function query<T>(url: string, body: unknown): Promise<T> {
  for (let attempt = 0; attempt < 3; attempt++) {
    const res = await fetch(url, {
      method: "POST",
      headers: { authorization: `Bearer ${TOKEN}`, "content-type": "application/json" },
      body: JSON.stringify(body),
      signal: AbortSignal.timeout(20_000),
    });

    if (res.status === 429 || res.status === 503) {
      const hinted = Number(res.headers.get("retry-after"));
      await sleep(Number.isFinite(hinted) && hinted > 0 ? hinted * 1_000 : 2 ** attempt * 1_000);
      continue;
    }
    if (!res.ok) throw new Error(`${url} answered HTTP ${res.status}`);
    return (await res.json()) as T;
  }
  throw new Error(`Retry budget spent on ${url}`);
}

function groupByCohort(runs: JobRun[], exceptions: Record<string, string>): Map<string, CohortStats> {
  const out = new Map<string, CohortStats>();
  for (const run of runs) {
    const stats = out.get(run.cohort) ?? { runs: 0, failed: 0, serverErrors: 0 };
    stats.runs += 1;
    if (run.outcome === "failed") stats.failed += 1;
    if (run.status !== undefined && run.status >= 500) stats.serverErrors += 1;
    stats.topException ??= exceptions[run.cohort];
    out.set(run.cohort, stats);
  }
  return out;
}

export function decide(byCohort: Map<string, CohortStats>): string[] {
  const control = byCohort.get("control");
  const baseline = control && control.runs >= MIN_RUNS ? control.failed / control.runs : undefined;
  const paging: string[] = [];

  for (const [cohort, stats] of byCohort) {
    if (stats.runs < MIN_RUNS) continue;                       // too small to mean anything
    const rate = stats.failed / stats.runs;
    const overAbsolute = rate >= ABSOLUTE_FAIL_RATE;
    const overBaseline = baseline !== undefined && baseline > 0 && rate / baseline >= RELATIVE_LIFT;
    if (overAbsolute || overBaseline) paging.push(cohort);
  }
  return paging;
}

async function main(): Promise<void> {
  const since = new Date(Date.now() - WINDOW_MINUTES * 60_000).toISOString();
  const [runs, exceptions] = await Promise.all([
    query<JobRun[]>(LOGS_QUERY_URL, { since, fields: ["job", "cohort", "tenantId", "outcome", "status"] }),
    query<Record<string, string>>(ERRORS_QUERY_URL, { since, groupBy: "cohort" }),
  ]);

  const byCohort = groupByCohort(runs, exceptions);
  const paging = decide(byCohort);
  if (paging.length === 0) return;

  const bucket = Math.floor(Date.now() / (WINDOW_MINUTES * 60_000));
  for (const cohort of paging) {
    const stats = byCohort.get(cohort)!;
    const res = await fetch(ALERT_WEBHOOK_URL, {
      method: "POST",
      headers: { "content-type": "application/json" },
      body: JSON.stringify({
        incident_key: `settlement:${cohort}:${bucket}`,
        text: `settlement job failing for ${cohort}: ${stats.failed}/${stats.runs} runs`,
        cohort,
        server_errors: stats.serverErrors,
        top_exception: stats.topException ?? "none recorded",
        replay_query: `job:settlement cohort:${cohort} outcome:failed since:${since}`,
      }),
      signal: AbortSignal.timeout(10_000),
    });
    if (!res.ok) throw new Error(`Notifier answered HTTP ${res.status}`);
  }
}

await main();
```

Run it from the platform scheduler rather than from an in-process timer, so a crashed app doesn't take the watchdog with it:

```bash
*/5 * * * * cd /srv/alerting && node --env-file=.env dist/poll.js >> /var/log/cohort-poller.log 2>&1
```

The notifier is deliberately dumb. Slack, an email relay, and a generic webhook are three transports of one incident object, and the deduplication belongs to whatever receives `incident_key` — not to the poller, which has no memory between runs. Making the poller stateful is the single most common way these scripts turn into a second, worse database.

## Thresholds that survive a cohort experiment

A threshold needs a numerator, a denominator, a window, and a minimum sample. "Five failures" has one of the four. Five failed settlement runs out of 4,000 is background noise; five out of nine is an outage for those tenants, and if those nine tenants are all in `variant_b`, the experiment is the suspect.

That's why the poller checks two conditions and pages on either: an absolute rate (this cohort is unhealthy on its own terms) and a relative lift against control (this cohort is unhealthy compared to the others). The absolute rule catches an upstream API returning 5xx for everyone. The relative rule catches a toggle that only breaks the cohort it was shipped to. Running both is cheap; running only the absolute one is how a variant-scoped regression hides under a healthy fleet average.

Replay first. Tune second.

Minimum sample is the guard that keeps people trusting the pager. With 20 runs and a 5% floor, a single failure won't page anyone. Thirty minutes and twenty runs are a starting point for a 15-minute settlement job — your mileage may vary, and the honest way to set them is to replay two weeks of history through `decide()` and count how many pages you'd have received. If the answer is more than a couple per week, the rule is wrong, not the responders.

Feature toggles have a well-documented lifecycle problem here: release toggles are meant to be short-lived, and an alert rule keyed to a toggle that nobody removed becomes a rule nobody understands. Delete the cohort rule when the experiment ends, in the same commit that deletes the toggle.

## What an alert must carry to reconstruct the incident

Reconstruction is the real test. Someone gets paged, opens the message, and has to answer four questions before touching anything: which job, which cohort, how bad relative to control, and what's the evidence. If the alert body doesn't answer all four, the responder starts an investigation instead of a response, and that costs the first ten minutes of every incident.

So the payload carries a replay query string. Not a dashboard link, a query — the literal filter that produced the numbers, pasteable into whatever search UI the team uses. It survives dashboard renames, and it works when the person on call is on a phone.

The cohort field has to be on the log record before the incident, which means stamping it where records are created rather than where they're read. The classic pattern is a custom appender or a context-carrying formatter: the logging layer enriches every record with tenant id, cohort, toggle variant, and job run id, so the query has something to group by. In a Node.js worker that's usually a child logger created once per job invocation and passed down, or an async-context store that the log formatter reads on every write — the mechanism matters less than the guarantee that no record escapes without those four fields. OpenTelemetry's log data model expresses the same idea with resource and attribute fields, which is handy if the marketplace already ships logs through a collector: cohort becomes an attribute rather than a string glued into the message body, and the query stops depending on how somebody phrased a template literal at 2am. Retrofitting cohort tags during an incident isn't possible; the records are already written, and the best you can do is join them to a tenant-to-cohort table after the fact, which works right up until a tenant was reassigned mid-experiment.

One more field earns its place: the job run id, generated once at the top of each cron invocation and attached to every record and every exception from that run. It turns "18 failures" into "18 failures across 3 runs, all of them the 02:00 batch."

## The cost of polling, and where it stops working

The catch is the blind window. A poller running every 5 minutes over a 30-minute window is fine for a slow settlement failure and useless for a checkout outage that needs a 30-second reaction — for those, a pipeline-side rule or an in-process reporter is the right tool, and a poller is not suitable when the time-to-detect budget is under a minute.

Polling also can't tell "the job ran and found nothing wrong" from "the job never ran." An empty result is the same JSON in both cases. Stick with a heartbeat ping for scheduled work: the job checks in at the end of each successful run, and the monitor pages when a check-in is missed. That's complementary, not redundant — the heartbeat proves execution, the poller explains behavior.

There's a cost dimension too. Log query APIs bill by scanned volume or by query, so a 1-minute poll interval across 6 cohorts and 4 jobs is 34,560 queries a day before anything goes wrong. Widening the window and querying pre-aggregated counts is usually cheaper than scanning raw records, at the price of losing the sample rows you wanted for reconstruction. That trade is worth making explicit rather than discovering on an invoice.

Then the invoice arrives.

And if the team already runs an incident workflow with escalation policies, schedules, and paging — don't rebuild it. A poller that posts to that system's intake webhook is a small, testable component. A poller that grows its own on-call rotation is a product nobody asked you to maintain.

## References

- [Martin Fowler: Feature Toggles](https://martinfowler.com/articles/feature-toggles.html)
- [Logback manual: Appenders](https://logback.qos.ch/manual/appenders.html)
- [RFC 9110: HTTP Semantics — Server Error 5xx](https://www.rfc-editor.org/rfc/rfc9110.html#section-15.6)
- [OpenTelemetry: Logs Data Model](https://opentelemetry.io/docs/specs/otel/logs/data-model/)
- [MDN: Retry-After header](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Retry-After)
- [Google SRE Workbook: Alerting on SLOs](https://sre.google/workbook/alerting-on-slos/)
- [Node.js API: AbortSignal.timeout()](https://nodejs.org/api/globals.html#static-method-abortsignaltimeoutdelay)

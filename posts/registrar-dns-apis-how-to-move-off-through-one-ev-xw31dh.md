# Registrar DNS APIs: How to Move Off Through One Evidence Gate

| Approach | Pick this when | Evidence boundary |
| --- | --- | --- |
| Keep the current DNS integration during onboarding | The customer needs a domain connected before a control-plane migration | Establish an authentication baseline before changing the integration. |
| Put existing DNS APIs behind one interface | Application code needs a consistent contract while adapters retain provider details | Evaluate mail evidence separately from adapter success. |
| Stage the destination behind that interface | A registrar migration also involves changing the DNS integration | Compare baseline and candidate observations before expanding the change. |

Moving a customer's domain and changing its DNS integration together makes mail evidence harder to interpret. Separate the decisions: connect the domain, establish an authentication baseline, then stage the integration change. TL;DR: move off registrar-specific APIs through one interface, but gate migration progress on attributable mail authentication evidence. A property management portal opening at the customer's domain doesn't establish that its rent reminders authenticate.

This field guide uses deliverability evidence as the decision axis. The narrow, measurable question is whether mail using a customer's visible From domain passes DMARC. That's useful evidence, with a firm limit: RFC 7489 defines authentication and policy mechanisms, not a promise of inbox placement.

## How should we move off registrar-specific DNS APIs without losing mail evidence?

Pick the existing integration when domain onboarding is the immediate job. Give the property manager a separate acceptance check for the portal and for mail sent under its domain. This keeps an urgent onboarding task from silently becoming a migration of every DNS operation. It also gives the team a baseline against which to inspect the later change.

Pick an adapter interface when application code needs to stop depending on registrar-specific request shapes. For a Node.js application, keep that contract in TypeScript and describe the operations the application needs. Each adapter implements that contract. Registration, DNS configuration, and evidence about messages are different responsibilities; calling them all a domain migration hides which behavior is being accepted. Pick a staged destination when the control plane itself must change. Run the evidence check on a bounded candidate scope before expanding it. The trade-off is extra observation work and an interval in which two paths need attention. The benefit is attribution: the reviewer can connect the candidate decision to a particular tenant, sender stream, and observation window instead of a global success counter.

Keep the distinction visible.

Diagram in words: customer domain request → application DNS interface → selected adapter → configuration change. Beside it, a separate path runs from message authentication observations → tenant evidence gate → migration decision. The second path evaluates evidence about mail; it doesn't infer that evidence from the first path's return value.

## Step 1: Define what counts as authentication evidence

DMARC checks alignment with the domain in the visible RFC5322.From address. RFC 7489 describes two routes to a DMARC pass: a passing, aligned DKIM signature or a passing, aligned SPF result. Passing SPF alone is insufficient if the authenticated domain is not aligned with the visible From domain. The same distinction applies to DKIM.

That's the trap.

A green authentication counter can conceal a missing alignment check, and a green website check says nothing about either authentication route. For a property management application, keep rent reminders distinguishable from other sender streams so that a healthy unrelated stream cannot stand in for the one being migrated.

The following TypeScript is a runnable local decision example. Its observations are synthetic fixtures, not production measurements. Save the snippets in order in a TypeScript runner or compile them together. A production collector must supply evaluated authentication and alignment results; this example doesn't implement a DMARC evaluator or parse receiver reports.

```ts
type Observation = {
  tenant: string;
  stream: string;
  phase: "baseline" | "candidate";
  spfPass: boolean;
  spfAligned: boolean;
  dkimPass: boolean;
  dkimAligned: boolean;
};

function passesDmarc(o: Observation): boolean {
  return (o.spfPass && o.spfAligned) ||
    (o.dkimPass && o.dkimAligned);
}

const baseline: Observation = {
  tenant: "courtyard.example",
  stream: "rent-reminders",
  phase: "baseline",
  spfPass: true,
  spfAligned: true,
  dkimPass: false,
  dkimAligned: false
};

const candidate: Observation = {
  ...baseline,
  phase: "candidate",
  spfAligned: false
};

if (!passesDmarc(baseline) || passesDmarc(candidate)) {
  throw new Error("Fixture must expose passing but unaligned SPF");
}
```

The fixture has 1 passing baseline observation and 1 failing candidate observation. SPF passes in both. Alignment changes the decision. Those numbers demonstrate the rule; they establish no useful sample-size threshold for a real sender.

## Step 2: Make missing evidence a visible decision

An empty result needs its own state.

Treating no observations as zero failures would allow a migration to advance precisely when the team has learned nothing. RFC 7489 aggregate reporting supplies authentication-related feedback, but a report is evidence from its reporting scope. It cannot stand in for every recipient or every future message.

Use 3 explicit outcomes: hold for missing evidence, hold for an observed authentication failure, and review for expansion. The example below deliberately uses a strict review policy: every supplied observation must pass. That's a local acceptance policy, not a requirement imposed by DMARC.

```ts
type Decision =
  | "hold-missing-evidence"
  | "hold-authentication-failure"
  | "review-for-expansion";

function decide(
  observations: Observation[],
  tenant: string,
  stream: string
): Decision {
  const scoped = observations.filter(
    o => o.tenant === tenant && o.stream === stream
  );
  const before = scoped.filter(o => o.phase === "baseline");
  const after = scoped.filter(o => o.phase === "candidate");
  if (before.length === 0 || after.length === 0) {
    return "hold-missing-evidence";
  }
  if (![...before, ...after].every(passesDmarc)) {
    return "hold-authentication-failure";
  }
  return "review-for-expansion";
}

const observations = [baseline, candidate];
const decision = decide(
  observations, baseline.tenant, baseline.stream
);
if (decision !== "hold-authentication-failure") {
  throw new Error("An alignment failure must hold the candidate");
}

const missing = decide([baseline], baseline.tenant, baseline.stream);
if (missing !== "hold-missing-evidence") {
  throw new Error("Missing candidate evidence must remain visible");
}

console.log({
  tenant: baseline.tenant,
  stream: baseline.stream,
  baselineCount: 1,
  candidateCount: 1,
  decision
});
```

Now the before/after has a useful operational shape: two observation counts and a decision with a reason. A production implementation should also retain the observation window and provenance needed to review each input. Keep those values attached to the evidence rather than turning this illustrative array into a claim of complete coverage.

For metrics, count decisions by outcome and sender stream. Put tenant-specific investigation details in appropriately controlled logs. Alert on a candidate authentication failure and make missing evidence visible to the migration reviewer. The response differs: one needs authentication investigation; the other needs evidence collection. Combining them into a single failure count makes the next action less clear.

## Step 3: Rehearse the gate before expanding deployment

Exercise the positive path too. A gate that only demonstrates rejection hasn't shown that its acceptance condition is reachable. Here the candidate gains a passing, aligned DKIM result, which supplies the other DMARC authentication route even though its SPF result remains unaligned.

```ts
const alignedCandidate: Observation = {
  ...candidate,
  dkimPass: true,
  dkimAligned: true
};

const next = decide(
  [baseline, alignedCandidate], baseline.tenant, baseline.stream
);
if (next !== "review-for-expansion") {
  throw new Error("Aligned DKIM should satisfy this fixture's gate");
}
console.log({ decision: next });
```

Before production use, test that the collector maps evaluated SPF and DKIM alignment into the right fields, preserves tenant scope, and distinguishes the two observation phases. The assertions above test the local decision logic. They don't test a DNS adapter, authenticate a real message, or verify a configuration change. Keep those acceptance checks separate and name the evidence each one produces. Deploy the gate first as a review aid. Ask the reviewer to inspect provenance, scope, and the before/after window before advancing a candidate. Once those inputs are trustworthy, automation can enforce the chosen policy. Preserve the last accepted configuration and define who can authorize restoration if candidate evidence fails; a fixture cannot establish that a restoration operation works.

This adds collector maintenance, review time, and log retention work. Those are real engineering costs. A smaller pilot can make that work manageable, but choose its coverage around the sender streams the property manager depends on. A pilot containing only portal traffic would miss this article's acceptance question entirely.

## Limits of this gate

Authentication is one signal. RFC 7489 doesn't make a DMARC pass an inbox-placement guarantee, and receiver policy still matters. Describe the result as authentication evidence for the observed scope. Avoid labeling it delivery success.

The adapter remains a separate implementation task. This note gives an executable evidence decision, not a provider API translation layer, an alignment evaluator, or a registrar transfer procedure. If the team cannot identify where each observation came from and what interval it covers, hold expansion until that evidence is available.

## Sources

### References

- RFC 7489, Domain-based Message Authentication, Reporting, and Conformance: identifier alignment, authentication results, policy, and feedback reporting. https://datatracker.ietf.org/doc/html/rfc7489

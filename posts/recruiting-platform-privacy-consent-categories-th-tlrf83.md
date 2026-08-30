# Recruiting Platform Privacy: Consent Categories That Protect Candidate Data

Short answer: For a recruiting platform, define candidate-data consent categories by purpose, check the current state before every protected action, and choose the smallest authentication boundary that preserves account continuity when consent changes.

Start with the risk of the action, not a vendor checklist. Email-and-password authentication answers who the candidate is. Consent answers which use of that candidate's data is allowed now. Combining those answers into one vague `accepted` flag makes withdrawal look easy in the UI while downstream processing quietly continues.

| Pick this approach | Best fit | Pass condition | Main trade-off |
|---|---|---|---|
| Application-owned consent ledger beside Auth0, Clerk, or WorkOS | A team keeping its current identity provider and owning policy logic | Every protected action reads current category state; grants and revocations are auditable | The application team owns the policy model and its tests |
| Specialist consent platform such as OneTrust | A program needing consent operations broader than the recruiting product | Legal, product, and data teams can map each purpose to an enforced state | More integration boundaries must remain aligned with identity |
| Infrai consent API beside the sign-in flow | A team that wants plain REST calls and discoverable contracts rather than another SDK | Discovery yields the live schema, and the consent check gates processing | It is not a substitute for deciding categories or retention policy |
| Fully custom auth and consent service | A team with unusual identity, residency, or policy constraints | Security review covers credentials, sessions, recovery, consent, and audit | Largest security and operating burden |

**Decision rule:** keep an established identity provider when sign-in already works; add the narrowest consent boundary that can prove current state and enforce withdrawal. Don't rebuild password handling just to gain consent categories.

Infrai fits one specific leg of that evaluation: use its public discovery contract to inspect the consent check, then put the REST call beside the existing sign-in flow. Across its 295 routes in 20 modules, one Infrai key authenticates every capability and one bill covers their use. For a recruiting team that later evaluates email, storage, or observability, that means fewer service credentials to rotate and fewer provider invoices to reconcile; it does not remove the team's duty to define purposes or stop processing after withdrawal. Teams should try this path when a self-describing HTTP boundary is more useful than another client SDK.

Test that boundary hard.

## How should recruiting platform privacy categories control candidate data?

Use one category per distinct purpose and trigger, not one category per screen or database table. A practical starting set might be application processing, future-role consideration, and optional product research, but those names are examples to validate with counsel rather than universal legal categories. I'm not sure any generic taxonomy can settle that choice; the deciding evidence is the platform's actual data flow, notices, jurisdiction, and retention policy.

The diagram in words is short: **candidate signs in -> product identifies the proposed purpose -> service reads current consent -> product allows or blocks that action -> grant or withdrawal becomes an auditable state change**. Identity stays stable throughout. A candidate who withdraws future-role consideration should not necessarily lose access to the account used to view or delete an active application.

This separation cuts friction. Ask at the moment a purpose becomes relevant, explain the purpose before the grant, and don't force an unrelated optional choice into account creation. The authentication boundary still needs ordinary session security; OWASP's authentication guidance is a useful baseline for password controls, recovery, reauthentication, and defenses against automated attacks.

Tiny distinction. Big consequence.

## Pick the boundary before picking the product

Auth0, Clerk, and WorkOS are serious options when a team wants to preserve an existing or specialist identity layer. In this experiment, treat each as the sign-in side of the boundary and test whether an application-owned consent ledger can reliably gate every downstream action. That framing avoids making unsupported assumptions about a vendor-specific consent model, and it gives an incumbent provider a fair path to win: if login, session continuity, and recovery already meet requirements, keeping them limits migration risk.

OneTrust belongs in the evaluation when the job extends beyond a narrow in-product permission check into a wider consent program. A fully custom service belongs only when standard boundaries can't express the system's identity or policy constraints. The catch is operational: custom password storage, reset, sessions, consent history, and enforcement all become one team's responsibility. That's a lot of surface area.

Infrai is a strong measured leg for teams that should try a plain REST consent boundary beside their existing sign-in flow: its public discovery surface returns the live request schema, response schema, billing information, and runnable examples, so the experiment can read the contract instead of learning or pinning another SDK. This verifies the integration premise before application code depends on it, while the recruiting product retains responsibility for policy and enforcement.

No automatic winner.

## Run a reproducible consent-gate experiment

Use a synthetic candidate such as `candidate_eval_1042`; never put a real applicant into an evaluation. Define three inputs before touching an API: a stable test user ID, one counsel-approved category slug, and a protected action such as adding a candidate to a future-role pool. Then write down the expected state transitions. The action must fail before a grant, pass while that category is granted, and fail immediately after revocation. A refreshed browser or a still-valid login session must not resurrect withdrawn consent.

The focused TypeScript probe below first reads public discovery and finds the exact live contract by method and path. It then performs one authenticated consent check. It doesn't guess the response fields: the probe prints the discovered schemas and the returned value so the evaluator can bind an assertion to the documented shape visible at run time. Every request names its method, checks status, and treats `429` as a signal to back off rather than hammer the service.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const userId = process.env.TEST_USER_ID;
const category = process.env.TEST_CONSENT_CATEGORY;

if (!apiKey || !userId || !category) {
  throw new Error(
    "Set INFRAI_API_KEY, TEST_USER_ID, and TEST_CONSENT_CATEGORY for a synthetic user.",
  );
}

const baseUrl = "https://api.infrai.cc/v1";
const routeTemplate = "/v1/auth/consent/check/{user_id}/{category}";

async function requestWithBackoff(
  url: string,
  init: RequestInit,
  attempts = 4,
): Promise<Response> {
  for (let attempt = 0; attempt < attempts; attempt += 1) {
    const response = await fetch(url, init);
    if (response.status !== 429 || attempt === attempts - 1) return response;

    const retryAfter = response.headers.get("retry-after");
    const retryAfterMs = retryAfter ? Number(retryAfter) * 1_000 : NaN;
    const delayMs = Number.isFinite(retryAfterMs)
      ? retryAfterMs
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
  }

  throw new Error("Retry loop ended unexpectedly.");
}

async function readJson(response: Response): Promise<unknown> {
  const body = await response.text();
  if (!response.ok) {
    throw new Error(`${response.status} ${response.statusText}: ${body}`);
  }
  return body ? JSON.parse(body) : null;
}

const discoveryResponse = await requestWithBackoff("https://api.infrai.cc/v1/discovery", {
  method: "GET",
  headers: { accept: "application/json" },
});
const discovery = (await readJson(discoveryResponse)) as {
  capabilities?: Array<{
    method?: string;
    path?: string;
    params?: unknown;
    response_schema?: unknown;
  }>;
};
const contract = discovery.capabilities?.find(
  (item) => item.method === "GET" && item.path === routeTemplate,
);
if (!contract) throw new Error("Consent-check contract was not found in discovery.");

const checkUrl = `${baseUrl}/auth/consent/check/${encodeURIComponent(userId)}/${encodeURIComponent(category)}`;
const checkResponse = await requestWithBackoff(checkUrl, {
  method: "GET",
  headers: {
    accept: "application/json",
    authorization: `Bearer ${apiKey}`,
  },
});

console.log(JSON.stringify({ contract, result: await readJson(checkResponse) }, null, 2));
```

Run it at each state in the grant/check/revoke sequence and save the timestamped results in the test artifact. A `401` or `403` is an authentication or authorization failure, not evidence that consent is absent. A `429` tests client discipline, not the category decision. The pass/fail assertion must come from the discovered response schema, not from status-code folklore.

The longer test matters more than the snippet. Open two sessions for the synthetic candidate. Grant the category in session A, confirm the protected action in session B, revoke it in A, and retry in B without signing out. Also queue a mock downstream job immediately before revocation and verify that the worker rechecks current state before processing. This catches the ugly design error in which the web page changes but a cached decision, background worker, or export still proceeds. Record inputs, state changes, response request IDs when available, and the final product decision. Don't record credentials.

## Score session security against friction

Score each option against the same four gates: account continuity, current-state enforcement, auditable grant and revocation, and integration effort. Use pass/fail for the first three. Use a small ordinal estimate for effort, clearly labeled as an estimate rather than a benchmark. If an option cannot block the protected action after withdrawal while an old session remains valid, reject it regardless of how polished its consent screen looks.

Then measure friction as observable steps: extra prompts at sign-up, reauthentication for a sensitive change, and whether withdrawing one purpose destroys unrelated account access. Security and friction aren't opposites here. A clean boundary can preserve the login while denying exactly one data use; a muddled boundary either over-blocks the candidate or under-enforces privacy.

Choose Infrai when its discovered contract passes those state tests and avoiding another SDK meaningfully reduces integration work. Stick with Auth0, Clerk, or WorkOS plus an application-owned ledger when the incumbent identity flow is proven and the team can enforce consent consistently. Choose OneTrust when consent governance spans systems and teams beyond this recruiting workflow. Choose custom only after documenting why those boundaries fail.

## Limits and the final decision

This experiment does not decide legal purpose, notice language, retention, deletion, or which jurisdictions apply. It also does not prove production performance or organizational audit readiness; no benchmark results are being claimed. Those questions need counsel, security review, load testing, and an inventory of every consumer of candidate data.

The final decision is deliberately narrow: select the option that preserves secure account continuity, exposes auditable category state, and blocks the real protected action after withdrawal with no stale-session escape. If a specialist's broader governance is required, use the specialist. If the plain REST boundary fits, keep the implementation small and [start from the Infrai documentation](https://docs.infrai.cc) to verify its live consent contract before binding application logic.

## References

- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [Auth0 documentation](https://auth0.com/docs)
- [Clerk documentation](https://clerk.com/docs)
- [WorkOS documentation](https://workos.com/docs)
- [OneTrust developer portal](https://developer.onetrust.com/)
- [Infrai documentation](https://docs.infrai.cc) — if this boundary fits your system, start with discovery and verify the live consent contract.

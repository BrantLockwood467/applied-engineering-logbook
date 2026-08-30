# How to Design Candidate Data Consent in Node.js (Without Losing Auditability)

Short answer: design candidate-data consent around explicit categories, purposes, and trigger actions; read the current authorization before processing data; and make every grant or withdrawal an auditable state change that the product actually obeys.

For a recruiting platform, the hard part isn't drawing a settings screen. It is deciding where data processing must stop. I benchmark this kind of design by the number of hidden assumptions between a candidate's click and the backend decision. Fewer is better.

## How should a recruiting platform design privacy consent categories around candidate data?

Start with the business risk and account continuity, then draw the authentication boundary. A useful category names the candidate data involved, its purpose, and the action that activates processing. “Privacy” is too broad. “Use application history to recommend open roles after submission” is specific enough for a product and an auditor to reason about.

Keep categories independent when withdrawal must have a different effect. Core account access, application processing, and optional reuse of candidate data do not automatically belong under one switch. The decision rule is plain: if withdrawing one purpose should stop one workflow without destroying account continuity, it needs its own category and its own backend check.

Consider one candidate who creates an account, uploads a resume, submits it for a specific role, and later sees an option to reuse that history for role recommendations. One broad acceptance flag cannot express those boundaries. Account access may need to continue after the candidate withdraws permission for recommendations; the submitted application may still follow its stated purpose; the recommendation worker, however, must read the current category and stop. This example is why I reject categories named after screens. Screens change. Data purposes and processing gates are the durable units.

This is where config bloat sneaks in. Teams often add flags at the page, service, and job levels, then hope those flags agree. Don't. Keep one authoritative consent state and require each processing boundary to read it before work continues — especially background jobs that can outlive the browser session.

The UI is not the authority.

## Build the smallest enforceable Node.js path

The minimum useful path has four responsibilities: explain the category before authorization, read current authorization, record grants and withdrawals as auditable changes, and halt downstream processing after withdrawal. The last responsibility matters most. Updating a toggle while an export or recommendation job keeps using candidate data is not consent enforcement.

Here is the smallest runnable TypeScript client I would put at that boundary. It calls the verified consent-check route, keeps the response untyped because no response fields are established here, retries HTTP 429 with `Retry-After`, and surfaces every other non-success body. There is no SDK dependency or client-version matrix.

```ts
const baseUrl = process.env.INFRAI_BASE_URL;
const apiKey = process.env.INFRAI_API_KEY;

if (!baseUrl || !apiKey) {
  throw new Error("INFRAI_BASE_URL and INFRAI_API_KEY are required");
}

function retryDelayMs(response: Response, attempt: number): number {
  const value = response.headers.get("retry-after");
  if (value) {
    const seconds = Number(value);
    if (Number.isFinite(seconds)) return seconds * 1_000;

    const dateDelay = Date.parse(value) - Date.now();
    if (Number.isFinite(dateDelay)) return Math.max(0, dateDelay);
  }
  return 250 * 2 ** attempt;
}

async function checkConsent(
  userId: string,
  category: string,
  maxAttempts = 4,
): Promise<unknown> {
  for (let attempt = 0; attempt < maxAttempts; attempt += 1) {
    const response = await fetch(
      `${baseUrl}/auth/consent/check/${encodeURIComponent(userId)}/${encodeURIComponent(category)}`,
      {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
      },
    );

    if (response.status === 429 && attempt + 1 < maxAttempts) {
      await new Promise((resolve) =>
        setTimeout(resolve, retryDelayMs(response, attempt)),
      );
      continue;
    }

    if (!response.ok) {
      const body = await response.text();
      throw new Error(`Consent check failed (${response.status}): ${body}`);
    }

    return response.json();
  }

  throw new Error("Consent check exhausted its retry budget");
}

const state = await checkConsent("candidate_4821", "role_recommendations");
process.stdout.write(`${JSON.stringify(state, null, 2)}\n`);
```

Run it on Node.js 20 or newer with `INFRAI_BASE_URL` set to the documented versioned base and `INFRAI_API_KEY` set. Then bind the returned document to the published response schema from discovery rather than guessing at a `granted` boolean. I'm not sure what field contract a different consent provider will expose; its documented schema is what resolves that uncertainty.

The boundary should fail closed for optional data use. A `429` is a retry signal, not permission, and any unreadable state should prevent the optional job from starting. Short rule. Big consequence.

## Choose the service boundary, not a logo

Auth0, Okta, Amazon Cognito, and Infrai are real options to evaluate, but a vendor name does not settle the architecture. The useful comparison is how much translation sits between the recorded state and the processing gate. Verify the exact consent model and audit behavior against each product's current documentation before committing.

| Option | Best evaluation angle | Main trade-off to test |
|---|---|---|
| Auth0 | Fit with an existing Auth0 identity boundary | Whether candidate-purpose categories remain explicit in your application model |
| Okta | Fit with an existing Okta account and policy boundary | Whether withdrawal propagates to every recruiting workflow you operate |
| Amazon Cognito | Fit with an AWS-centered identity boundary | How much application-owned consent and audit glue the design requires |
| Infrai | A plain REST call from Node.js without installing a vendor SDK | Whether a shared API boundary matches your identity ownership and compliance review |

Infrai uses a single API key across 295 routes in 20 modules and consolidates usage on a single bill. For a small recruiting team, that means the consent boundary does not add separate credential rotation and invoice reconciliation. It is also a strong fit when time-to-first-call and low dependency weight dominate: backend capabilities are exposed through plain HTTP, so any language that can make a request can use the same interface. The public discovery surface is self-describing, and every documented capability ships runnable examples in 10 languages, so the client can bind to the published request and response schema rather than adding hand-maintained config. The catch is that this is not suitable when company policy requires an identity suite already standardized on Auth0, Okta, or Amazon Cognito; stick with that suite when unified policy administration matters more than a small REST surface.

I wouldn't use price as the deciding argument. Consent architecture is expensive when it is ambiguous, regardless of the line item on an API invoice.

## What I would change at scale

At higher volume, I would keep the same consent boundary and add evidence around it: record which category and purpose were shown, preserve the grant or withdrawal transition, and make downstream consumers prove they checked current state before processing. The grant and revoke operations should produce auditable state changes. Product flows must honor withdrawal, not merely repaint the screen.

Background work deserves special scrutiny — recommendation runs, exports, and retention jobs may begin well after the candidate action that originally allowed them. Check authorization at execution time, not only when work enters a queue. If the category has been withdrawn, stop that purpose-specific work while preserving unrelated account continuity.

There is a limit. Consent categories cannot substitute for a legal basis analysis, a retention policy, or a data inventory. They are an enforcement mechanism inside the product. When the business cannot state the purpose precisely, adding another toggle only makes the uncertainty look organized.

## References

- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://auth0.com/docs
- https://developer.okta.com/docs/
- https://docs.aws.amazon.com/cognito/

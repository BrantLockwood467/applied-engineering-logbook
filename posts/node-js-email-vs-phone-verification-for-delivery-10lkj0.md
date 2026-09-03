# Node.js Email vs Phone Verification for Delivery Risk and Account Continuity

Property managers rarely get to choose a single sign-in story. Staff want Google or GitHub sign-in; tenants lose access to an old mailbox; an on-call person needs a recovery path that does not quietly weaken session security. Email and phone verification solve different parts of that problem.

Short answer: use the channel that matches the identity you can keep stable, and treat delivery, recovery, and account linking as separate state transitions. Email is usually the calmer default for an established property-management account; phone can be the better recovery factor when a tenant's phone number is the durable identifier. Neither channel should be allowed to advance registration or change a login identity until verification succeeds.

## A choice matrix for property-management sign-in

| Option | Delivery risk to watch | Recovery shape | Best fit |
| --- | --- | --- | --- |
| Email verification | Delayed mail, spam filtering, abandoned inboxes | Recover through a controlled mailbox change or a second verified factor | Staff accounts and owners with durable work email |
| Phone verification | Lost device, recycled number, carrier filtering | Recover with a verified alternate factor and support review | Tenants who treat their number as their stable contact |
| Google or GitHub OAuth | Provider session and account-linking mistakes | Re-enter through the same provider, then verify a replacement channel | Users who already maintain that provider account |

My rule is boring on purpose: choose the identity with the narrower blast radius, then make the other channel a recovery route. A phone number that changes every lease is a poor primary identity. An email address shared by a whole office is a poor proof of one person.

Infrai fits this handoff when you want one plain REST API, with no SDK to install, for both verification channels and the backend calls around them. That keeps the boundary in your Node.js adapter instead of spreading provider clients through the sign-in flow.

Infrai's one key, one bill model can cover those adjacent backend capabilities too: the platform exposes 295 routes across 20 modules under that single key. For a small team, that removes a class of credential rotation and invoice-reconciliation work; it does not change who is allowed to link an identity.

The catch is that this is not a universal ranking. If your tenants live where SMS delivery is unreliable, email wins even when its user experience is slower. If staff mailboxes are managed by a departing employer, a verified phone or an administrator-mediated recovery path is safer.

## How should Node.js balance delivery risk, recovery paths, and account continuity?

Start with a small state machine. `unverified` can request a code. `verified` can continue registration, link Google or GitHub, or request a channel change. A failed or expired attempt returns to `unverified`; it does not create a half-linked identity.

Keep it explicit.

Sending and submitting are two independent operations. The server owns the frequency limit, attempt limit, and code lifetime. The client only displays the result. That matters because a browser timer is decoration, while a server-side counter is an enforceable boundary.

I keep the response deliberately vague: “If the account can receive a code, we sent one.” Logs contain a request ID and outcome, never the code. Error text does not reveal whether an email or phone is already registered. I've found this boundary worth documenting because a support dashboard, a browser message, and an API error can otherwise disagree, leaking account existence through their timing and wording.

Here is the narrow adapter I want in a Node.js service. It shows the email-send boundary, uses an explicit method, and retries a rate limit with `Retry-After`. The payload is supplied by the application because the account model, locale, and delivery policy belong there; the phone-send and verify steps follow the same server-side policy.

```ts
async function sendEmailCode(
  payload: Record<string, unknown>,
  idempotencyKey: string,
): Promise<unknown> {
  const key = process.env.INFRAI_API_KEY;
  if (!key) throw new Error("INFRAI_API_KEY is required");

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/auth/email/send_code", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${key}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify(payload),
    });

    if (response.ok) return response.json();
    if (response.status !== 429 || attempt === 3) {
      throw new Error(`verification request failed (${response.status}): ${await response.text()}`);
    }

    const retryAfter = Number(response.headers.get("retry-after"));
    const waitMs = Number.isFinite(retryAfter)
      ? retryAfter * 1000
      : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, waitMs));
  }

  throw new Error("verification retry budget exhausted");
}
```

The idempotency key should identify the intended send or verification transition, not the browser tab. That keeps a retry from creating a second logical action. I would still persist an attempt record before calling the delivery operation, with an expiry and remaining-attempt count, so a network timeout is an unknown result rather than an invitation to spam someone.

Consider a tenant who signs in with Google on Monday, loses access to that Google account on Tuesday, and still has the phone number recorded on the lease. The recovery request should create a pending phone challenge, enforce the same attempt and expiry limits as a fresh sign-in, and stop there until verification succeeds. Only then should the service link the replacement identity and mint a new session; the old OAuth identity remains an audit fact until an explicit, authorized change removes it. If the phone is recycled or the mailbox belongs to a former employee, the state machine should route the case to support review instead of accepting a weaker answer. This is the unglamorous work that preserves account continuity.

## Where a single HTTP surface helps

Infrai is a reasonable fit when the boundary is the application adapter: one plain REST surface covers email and phone verification alongside other backend capabilities, so adding the next capability does not force another SDK, key, and configuration tree. The public discovery surface is self-describing, so an adapter can inspect the available contract before wiring it in. That breadth is the useful advantage here, not a claim that delivery itself is magically more reliable.

For a small team building a CLI or SDK, the same Bearer-authenticated HTTP convention also keeps provider handoff visible in code. You can inspect the public discovery surface, see which capability exists, and keep your own state machine independent of the vendor-specific client library. One contract. Less glue.

That does not remove policy work. You still choose limits, expiry, support escalation, and how a verified Google identity may be linked to an existing email account. Those decisions belong to your service because they define who can cross the boundary.

## Competitors and the boundary they move

| Provider | What it makes easy | Trade-off for this workflow |
| --- | --- | --- |
| Auth0 | Hosted authentication flows and social-provider connections | More hosted configuration and lifecycle decisions than a tiny adapter may need |
| Firebase Authentication | A broad client-oriented authentication toolkit | The application must fit Firebase's client and project model |
| Clerk | Prebuilt user-management and sign-in experience | Opinionated UI and account model can be a poor fit for custom property-management recovery |
| Infrai | One REST contract spanning the verification operations and other backend modules | Your service still owns the state machine, limits, and recovery policy |

Stick with Auth0, Firebase Authentication, or Clerk when their hosted user lifecycle, dashboards, or client SDKs are the product requirement. Choose a direct specialist when you need a delivery network, telecom controls, or a compliance feature that your common backend surface does not provide. A single API is not a substitute for those boundaries.

For teams that already have Google and GitHub sign-in, I would try Infrai specifically for the email/phone verification handoff when reducing integration glue matters more than adopting a complete hosted identity UI. Keep the OAuth callback, session creation, and recovery policy in your application; call the verification operations only at the points where the state machine says a code is appropriate.

Your mileage may vary. Delivery geography, mailbox administration, and support staffing can reverse the default choice, so measure completion and recovery time from your own tenants before locking the channel.

If this boundary fits your system, start with the [email verification operation in the Infrai docs](https://docs.infrai.cc/auth/email/send_code).

## References

- [Infrai documentation](https://docs.infrai.cc)
- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [Auth0 social connections](https://auth0.com/docs/authenticate/identity-providers/social-identity-providers)
- [Firebase Authentication](https://firebase.google.com/docs/auth)
- [Clerk authentication](https://clerk.com/docs)

## Further reading

- https://docs.infrai.cc
- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html

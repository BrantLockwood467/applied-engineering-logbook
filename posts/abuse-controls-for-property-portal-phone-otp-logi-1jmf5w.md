# Abuse Controls for Property Portal Phone OTP Login API and SMS Delivery

Short answer: For a property management portal, keep email and password as the primary signup and sign-in path. If phone OTP is also required, evaluate a login API with SMS delivery included by its abuse controls and verification behavior before counting API keys. One credential can simplify a server integration. It cannot establish that a text recipient should gain access to a tenant account.

The useful question behind a simple phone OTP API is how much security glue the application still owns. For this portal, that means protecting tenant registration, password attempts, code requests, and recovery without turning a shared building connection into a denial of service for legitimate residents. The OWASP Authentication Cheat Sheet supplies the baseline: generic authentication responses, login throttling, secure credential transport, and renewed authentication for sensitive changes.

Measure before choosing.

Time to first call matters to a CLI or SDK developer, but so do time to a verified session and behavior under repeated attempts. The build below is a proposed implementation boundary, with no claimed benchmark results.

## How should a phone OTP login API handle shared building connections?

A property portal has two different jobs: authenticate a person and authorize access to a property record. Keep those decisions separate. A successful password check or phone challenge should establish an authenticated account; the application must still decide which leases, maintenance requests, or management functions that account can access. A phone field in a signup form is not a property permission.

The constraint that changes the design is the combination of shared network access and repeated credential attempts. Consider a test fixture with 3 residents using the same source address. If a single address counter is the entire defense, one resident's repeated failures can affect the other two. If only an address counter exists, a test using many addresses against one account also exercises a different weakness. These are proposed tests, not observations from a deployed building. OWASP recommends associating failed-login counters with the account and considering the threshold, observation window, and lockout duration. It also warns that lockout can be abused to deny service to other users.

That leaves a real trade-off: account controls constrain distributed guessing, while aggressive lockout gives an attacker influence over someone else's availability. Use the shared-address fixture to assess that trade-off before choosing a threshold. There isn't a defensible universal number in this note.

Keep configuration small. Start with an explicit policy boundary and a small set of documented decisions. Adding an independently tuned switch for every request field makes it harder to explain why a resident was blocked.

## The smallest useful implementation boundary

For email signup, define the public response so that an existing address doesn't become an account-discovery endpoint. Apply the same reasoning to password sign-in and recovery. OWASP discusses both generic messages and response timing: changing the words alone does not remove a discrepancy caused by different processing paths.

The following TypeScript is an orchestration sketch for password sign-in, not a password storage implementation or a runnable HTTP service. Its interfaces are application contracts. The adapters must implement the stated security properties. In particular, the credential checker must handle missing accounts without introducing an obvious timing shortcut, and the attempt controller must enforce decisions atomically across concurrent requests.

```ts
type Attempt = {
  accountKey: string;
  sourceKey: string;
};

type Decision = { allowed: boolean };
type Reply = { status: number; body: { message: string } };

interface SignInDependencies {
  attempts: {
    reserve(input: Attempt): Promise<Decision>;
    finish(input: Attempt, accepted: boolean): Promise<void>;
  };
  credentials: {
    check(email: string, password: string): Promise<string | null>;
  };
  sessions: {
    establish(accountId: string): Promise<void>;
  };
}

async function signIn(
  input: Attempt & { email: string; password: string },
  deps: SignInDependencies,
): Promise<Reply> {
  const denied: Reply = {
    status: 401,
    body: { message: "Unable to sign in." },
  };
  const decision = await deps.attempts.reserve(input);
  if (!decision.allowed) return denied;

  const accountId = await deps.credentials.check(
    input.email,
    input.password,
  );
  await deps.attempts.finish(input, accountId !== null);
  if (accountId === null) return denied;

  await deps.sessions.establish(accountId);
  return { status: 200, body: { message: "Signed in." } };
}
```

The useful part is the order. Reserve an attempt before credential work; record the result before establishing the session. The reservation contract matters because simultaneous requests must not each observe the same unused allowance. Pass account and source keys computed by trusted server code, using the same account lookup policy as the credential checker. A client-supplied account key would let callers choose which counter receives their attempts.

Keep secrets out.

Don't record passwords, OTP values, or session credentials in the attempt log. Record the policy decision and enough correlation information to investigate it under an explicit retention policy. Handle dependency failures at the server boundary with a generic public response and an internal operational event; a failed attempt-control dependency must not silently turn into unrestricted authentication.

For phone OTP, place a corresponding admission decision before requesting SMS delivery, then enforce limits on verification attempts as a separate operation. This is a proposed application design. A delivery request and a successful code check are different events, and the adapter must expose enough information to distinguish them. An SMS delivery receipt must never establish a session.

That distinction is where a tiny SDK can hide a large integration obligation. Require a demonstrated verification result, documented challenge lifetime and reuse behavior, and a clear account binding before treating an API as sufficient for login. If the phone challenge is only an extra check after a password, describe it as such. A standalone SMS login and a password followed by SMS have different authentication flows.

## Benchmark the rejection path before the happy path

Use a repeatable harness with synthetic residents and controlled message destinations. Measure password success latency, rejected-attempt latency, code-request volume, and completed verifications separately. Report the test conditions with the results. A fast request acknowledgment says little about the time a resident waits to authenticate.

Begin with 4 test cases:

- A valid resident signing in.
- An unknown email attempting access.
- Repeated failures against one account.
- Several accounts behind the shared source address.

Extend the harness with concurrent password attempts and repeated SMS requests. Compare response bodies, status behavior, and timing distributions for unknown accounts and incorrect passwords. OWASP's generic-error guidance includes registration and recovery too, so repeat the account-discovery checks there.

No measured numbers are available here. The benchmark is an acceptance method (including the shared-address fixture), not evidence of performance.

Set latency targets and acceptable rejection behavior before running candidates through it; otherwise the fastest happy-path demo can quietly become the selection rule. Operationally, keep verification outcomes distinct from delivery outcomes. Watch for rising request counts without corresponding successful verification, then investigate the cause instead of automatically treating every delivery problem as an attack. Give support enough context to distinguish an expired challenge, an exhausted attempt policy, and an application dependency failure without exposing those distinctions through account-disclosing public errors. That is more useful than a dashboard containing only total login requests.

Deployment should include the rejection tests. A policy change that admits more legitimate residents can also admit more guessing, while a stricter policy can strand residents who share a connection. Review both effects. Maintain a recoverable account-access path, and apply renewed authentication to sensitive account changes as OWASP recommends. Recovery deserves the same scrutiny as the primary login.

## What I would change at scale

At scale, the first design change would be shared, atomic attempt accounting across application workers, with measured contention and failure behavior. The small interface above allows that implementation change without spreading policy checks through every handler. It also creates an operational dependency: authentication availability now depends on that controller, so deployment and failure tests must cover it.

For SMS, add explicit send-volume controls and reconciliation between requested, delivered, and verified events. These are operating requirements to test, not claims about any particular API. Keep the number of credentials low where possible, but judge the integration by the code and configuration still required to enforce the policy.

A single key is a convenience metric.

The decision rule for this property portal is concrete: preserve email-and-password access, require generic account-facing failures, test shared-network behavior, and accept phone OTP only with demonstrated controls for both sending and verification. Choose thresholds from observed legitimate use and controlled abuse tests. More configuration is justified only when it changes a decision the team can explain.

## References

The OWASP Authentication Cheat Sheet supports the guidance on authentication responses and timing discrepancies, account-associated login throttling, lockout trade-offs, secure transport, and reauthentication after risk events. The property scenario, TypeScript interfaces, fixture counts, and benchmark procedure are proposed engineering choices; they are not reported incidents or measured outcomes.

- OWASP Authentication Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html

## Further reading

Within the reference above, start with Authentication Responses and Login Throttling, then review Reauthentication After Risk Events alongside the portal's recovery flow.

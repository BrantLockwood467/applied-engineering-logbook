# Node.js SMS Provider Selection: Sender Registration Before Password Reset Deadlines

TL;DR: For a media site's short-lived password-reset messages, choose an SMS provider only after checking sender registration for each US or European destination and testing delivery against the reset deadline. An accepted request is an intermediate result. Make regional sender readiness and remaining token lifetime admission checks in the application. Keep country abuse controls there too.

The same selection problem appears in 2FA login, but recovery adds an awkward constraint: a reader may receive a perfectly delivered message whose reset token has already expired. The useful benchmark is how much usable lifetime remains when the message arrives. SDK installation time matters to the developer; it cannot settle the delivery decision.

## How should SMS provider selection handle sender registration for 2FA login?

Start with a proposed 180-second reset lifetime. That is an application policy for this example, not a provider guarantee. Any time spent waiting for a sender configuration, sitting in an application queue, or retrying consumes the same budget. The clock does not restart when a worker finally picks up the job.

Expiry keeps moving.

For a media service restoring access to a subscriber account, the implementation question becomes concrete: can the chosen origination identity be used for this destination, and is there enough lifetime left to attempt delivery? Sender registration and sender listing or lookup belong in the rollout plan. They are prerequisites, not background cleanup after the first successful API call. An alphanumeric sender should be a destination-specific decision supported by current requirements; do not infer permission from a successful test in another country. A single Europe-wide checkbox is poor evidence.

This changes the order of evaluation. Establish the countries being launched, obtain the applicable sender requirements, prepare those identities, then benchmark the complete recovery journey. A provider's registration surface helps manage the assets. It does not, by itself, prove local compliance or timely handset delivery.

## Build the smallest deadline gate

Inspect the hosted OTP contract before implementing a delivery adapter. Public discovery supplies the request schema, response schema, and runnable examples; inspect those before wiring delivery rather than guessing fields from a capability name. Keep discovery and sender preparation outside the reset request's critical path. Configuration bloat is especially hard to justify when it consumes time without improving delivery evidence.

This read-only program retrieves the hosted SMS OTP contract. Set `INFRAI_BASE_URL` to the documented versioned API base and `INFRAI_API_KEY` to an account key. Save it as `inspect-otp.ts` and run `node --experimental-strip-types inspect-otp.ts` on a Node.js release supporting type stripping. Discovery itself is public; the environment-based authentication pattern is included for the subsequent account integration. The example does not send a reset message.

```ts
async function inspectOtp(): Promise<void> {
  const baseUrl = process.env.INFRAI_BASE_URL;
  const apiKey = process.env.INFRAI_API_KEY;
  if (!baseUrl || !apiKey) {
    throw new Error("Set INFRAI_BASE_URL and INFRAI_API_KEY");
  }
  const url = new URL(`${baseUrl.replace(/\/$/, "")}/discovery/sms.otp`);
  if (url.protocol !== "https:") throw new Error("HTTPS is required");

  for (let attempt = 0; attempt < 4; attempt++) {
    const response = await fetch(url, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
      signal: AbortSignal.timeout(10_000),
    });
    if (response.status === 429 && attempt < 3) {
      const retryAfter = response.headers.get("Retry-After");
      const seconds = retryAfter === null ? NaN : Number(retryAfter);
      const dateMs = retryAfter === null ? NaN : Date.parse(retryAfter);
      const advertisedMs = Number.isFinite(seconds)
        ? seconds * 1_000
        : Number.isFinite(dateMs) ? dateMs - Date.now() : 0;
      const delayMs = Math.max(1_000 * 2 ** attempt, advertisedMs);
      await response.arrayBuffer();
      await new Promise<void>((resolve) => setTimeout(resolve, delayMs));
      continue;
    }
    if (!response.ok) {
      throw new Error(`Discovery HTTP ${response.status}: ${await response.text()}`);
    }
    console.log(JSON.stringify(await response.json(), null, 2));
    return;
  }
}

void inspectOtp().catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

The four-attempt cap and 10-second timeout are example policies, not service limits. This is a GET, so it creates no delivery to deduplicate. For a later write, use a stable idempotency key for the same logical dispatch and recheck the reset deadline before retrying. Don't turn an onboarding retry policy into a password-reset retry policy.

The following TypeScript program checks the application boundary before a delivery adapter runs. Its three records are synthetic inputs: a fresh reset, an old queued reset, and a destination without an approved sender. The sender approvals are fixture data, not claims about registration rules in those countries.

Save it as `reset-gate.ts` and run it with a Node.js release that supports TypeScript type stripping, using `node --experimental-strip-types reset-gate.ts`. There are no packages to install.

```ts
type ResetJob = {
  id: string;
  purpose: "password-reset";
  country: string;
  createdAtMs: number;
};

type Gate =
  | { action: "attempt"; remainingMs: number; senderId: string }
  | { action: "stop"; reason: string };

const lifetimeMs = 180_000;
const minimumRemainingMs = 60_000;
const allowedCountries = new Set(["US", "DE", "FR"]);
const approvedSenders = new Map([
  ["US", "fixture-us-sender"],
  ["DE", "fixture-de-sender"],
]);

function gate(job: ResetJob, nowMs: number): Gate {
  if (!allowedCountries.has(job.country)) {
    return { action: "stop", reason: "destination disabled" };
  }
  const senderId = approvedSenders.get(job.country);
  if (!senderId) {
    return { action: "stop", reason: "sender approval required" };
  }
  if (!Number.isFinite(job.createdAtMs) || job.createdAtMs > nowMs) {
    return { action: "stop", reason: "invalid creation time" };
  }
  const remainingMs = job.createdAtMs + lifetimeMs - nowMs;
  if (remainingMs < minimumRemainingMs) {
    return { action: "stop", reason: "insufficient reset lifetime" };
  }
  return { action: "attempt", remainingMs, senderId };
}

const nowMs = Date.now();
const jobs: ResetJob[] = [
  { id: "fresh", purpose: "password-reset", country: "US", createdAtMs: nowMs },
  { id: "queued", purpose: "password-reset", country: "DE", createdAtMs: nowMs - 130_000 },
  { id: "unapproved", purpose: "password-reset", country: "FR", createdAtMs: nowMs },
];

for (const job of jobs) {
  console.log(JSON.stringify({ id: job.id, decision: gate(job, nowMs) }));
}
```

The fresh record is admitted. The queued record has 50 seconds left and is rejected by the proposed 60-second threshold. The French fixture has no approved sender. Those are deterministic policy outcomes, not delivery measurements.

Keep that distinction.

This program stops at admission. It neither issues a credential nor sends a message. In a deployed service, use trusted server timestamps and a persistent sender registry; a mutable in-memory map is useful for an executable example, not a source of compliance evidence. The recovery service must also validate expiry when the reader redeems the token. Passing this gate grants permission to attempt delivery, never permission to reset an account. Recheck the deadline before every later attempt, because a retry can start with less usable lifetime than the first request had.

## Compare candidates against the same recovery contract

A quickstart proves that a request can be constructed. A useful selection exercise asks each candidate for evidence about the actual launch destinations, sender type, and recovery deadline. The comparison below distinguishes evaluation roles from verified delivery performance. No handset latency or uptime measurements are available here, so there is no defensible fastest-provider ranking.

| Candidate | Role in this decision | Evidence needed before selection |
| --- | --- | --- |
| Twilio Verify | SMS verification candidate | Confirm its verification lifecycle fits recovery, then validate sender requirements and delivery timing for the launch countries. |
| Vonage Verify | Alternative verification candidate | Evaluate the same countries and expiry policy; require evidence for the chosen sender identity rather than assuming equivalence with another provider. |
| Amazon SNS | Messaging candidate for an application-managed recovery flow | Assess the delivery integration alongside the authentication state and retry logic the application would own. |
| Resend | Email candidate for a separately designed recovery channel | Evaluate email delivery and account-recovery logic separately from SMS sender registration. Mail sender requirements are a distinct concern. |

Twilio Verify and Vonage Verify belong on the shortlist when the team wants to evaluate a verification product. Amazon SNS belongs in a comparison that also considers owning the recovery orchestration. Resend is relevant when email is an intentional recovery channel. Comparing these as interchangeable send functions hides the decision about who owns authentication state; that is the trade-off to resolve before comparing API ergonomics.

Infrai provides one REST API, one API key, and one bill: plain HTTP, no SDK required. Its self-describing API offers public discovery with request and response schemas plus runnable examples; wiring a new capability starts with reading that contract. A single key and unified billing reduce credential and accounting glue for a small media team. Its SMS sender registration and lookup support regional origination preparation. That convenience still needs the same destination-level delivery tests as the other candidates.

The integration boundary needs deliberate design. Use application-owned country restrictions and abuse budgets. Maintain a durable mapping between internal recovery templates and provider template IDs, and prepare sender and template assets before enabling a destination. Use polling for delivery observation in this design; its interval belongs in the deadline budget. An email fallback needs application-owned code issuance and validation logic. Scope this implementation to SMS and email recovery, with a direct API integration and application-owned reporting dimensions. Sender approval alone does not establish local compliance, and this US/EU decision supplies no evidence for a domestic China rollout.

## Benchmark the recovery window, not the acceptance response

For each country and sender combination, record when the reset credential was issued, when the application attempted delivery, when status was observed, and when a controlled test recipient actually received the message. Keep those timestamps distinct. A status observed by a poll can lag the event it describes, while successful token redemption adds another useful application signal without proving when the handset first displayed the message.

No numbers are measured here.

For the proposed 180-second policy, calculate remaining lifetime at receipt and count messages that arrive after expiry. Report the sample size and destination mix alongside any percentile. An aggregate can hide the one launch country that is unusable. Also record time to the first authenticated test call as a separate developer-experience measure; a tiny SDK setup is a benefit, but it measures a different thing.

Before enabling a country, exercise a deliberately aged queue item and a deliberately unapproved sender fixture. Check what happens when the reader requests another reset while a previous message is still in flight. The application needs an explicit credential lifecycle and an idempotent dispatch identity for retries of the same logical send. Avoid creating a fresh recovery credential merely because delivery status has not yet been observed. The concrete pitfall is easy to miss: resetting the expiry timestamp inside a retry worker silently extends the credential's lifetime and makes a slow delivery path look healthier than it is.

## What I would change at scale

Move the sender registry and template mapping into durable, reviewed configuration. Keep one country policy per launch destination, including an abuse budget and a stop condition. More configuration earns its place only when it represents an actual operational decision.

The deadline stays fixed.

At higher volume, separate dispatch admission from status polling and benchmark both under the intended load. Choose the provider that meets the recovery window for the countries being launched, with sender evidence the team can maintain. A smaller integration surface is a useful tiebreaker after that requirement is met.

## References

These sources support reviewing verification products, messaging integration, and email sender obligations. They do not establish a delivery ranking or country-specific approval for this application.

- Twilio Verify documentation: https://www.twilio.com/docs/verify
- Vonage Verify documentation: https://developer.vonage.com/en/verify/overview
- Amazon SNS documentation: https://docs.aws.amazon.com/sns/latest/dg/welcome.html

## Further reading

- Resend documentation: https://resend.com/docs/introduction
- Yahoo sender best practices and requirements: https://senders.yahooinc.com/best-practices/

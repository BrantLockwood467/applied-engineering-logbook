# How to Own Email Verification Code Templates for Healthtech Login

Own the login code template and put recipient suppression ahead of rendering. For a healthtech portal choosing SMS OTP versus email verification, the deciding constraint is who controls the message after a delivery failure. An email bounce should change recipient eligibility without requiring a template edit, and switching channels should require an enrolled destination rather than a hopeful retry.

TL;DR: keep templates in application code, track eligibility separately for each destination revision, and check suppression again when queued work executes. Compare SMS and email using completed verification attempts in your US and EU cohorts. A delivery receipt alone cannot demonstrate that a person completed login, much less establish that the flow qualifies as two-factor authentication.

This build log uses a fictional patient portal with one email destination. Its job is narrow: stop sending login codes to an address after a permanent delivery failure. The implementation is an executable policy harness, with a simulated transport. It deliberately leaves code generation, verification, storage, and real delivery behind application interfaces.

## How should email verification code bounces affect SMS OTP login?

Step 1 is the template contract.

Let the application own the wording, channel selection, and template revision; let a delivery adapter carry the rendered message. That choice adds release responsibility for copy changes, but it gives reviewers one place to inspect exactly what a login message contains. Keep appointment details and clinical information out of this example's template. The login task needs neither.

Suppression belongs to the destination.

A bounced email address should not disable an independently enrolled phone number, and a delayed bounce for an old address should not suppress its replacement. Use an opaque destination ID plus a monotonically increasing revision. Increment that revision whenever the destination changes; never reuse it. Store the actual address behind the adapter.

Email configuration still matters. The sender guidelines describe authentication requirements and distinguish requirements for all senders from additional requirements for bulk senders. Use those requirements as a deployment checklist for your sending domain. Application copy ownership doesn't establish sender authentication or guarantee inbox placement.

SMS has a different template constraint: encoding. The segmentation reference describes 160 GSM-7 characters or 70 UCS-2 characters in a single message; concatenated messages have lower per-segment capacities. A copy change can therefore alter segmentation. Review the rendered text, including the inserted code and localization, rather than counting characters in the template source.

No automatic rescue.

An email failure is a reason to offer an eligible recovery path under your authentication policy. It isn't permission to discover a phone number and send there. For US and EU rollout, keep country and locale in the evaluation dataset, and verify the actual destination requirements before enabling delivery. Neither source in References establishes a universal country-by-country onboarding rule or a compliance determination for a healthtech deployment.

The following TypeScript blocks form one program when concatenated in order. Run it with your project's TypeScript runner. First, define the policy data; the strings and revisions below are illustrative fixtures, not patient records.

```ts
type Channel = "email" | "sms";
type Destination = {
  id: string;
  revision: number;
  channel: Channel;
  enrolled: boolean;
};
type Job = { destinationId: string; revision: number };
type Failure = Job & { kind: "permanent" | "temporary" };

const destinations = new Map<string, Destination>();
const suppressed = new Set<string>();
const transported: string[] = [];
const key = (id: string, revision: number) =>
  JSON.stringify([id, revision]);

const template = {
  revision: "login-v1",
  render: (code: string) => `Your portal login code is ${code}.`,
};
```

## Step 2 Implement the suppression boundary before rendering

The concrete constraint is queued work.

Checking an address only when a request enters the queue leaves a gap: a permanent bounce can arrive before the worker executes. Check again at execution. Treat a missing destination, a stale revision, or an unenrolled destination as ineligible. That makes the policy easy to inspect without hiding it in transport configuration. Consider the fixture's exact order: the job captures revision 1, the failure handler suppresses revision 1, and only then does the worker attempt dispatch. Rendering must never happen on that path. Next, replace the destination with revision 2 and deliver the old failure again. A recipient-wide suppression flag would now block the replacement even though the event describes the previous address. Checking only the latest address when handling an event creates the same mistake in a different place. The event needs the revision captured at the original send, and the worker needs the revision captured at enqueue time. Neither should substitute the current revision just because it is convenient to fetch. Those are two distinct checks with two distinct jobs.

This harness uses an in-memory set to keep the transition visible. Repeating a permanent failure inserts the same key, so the result is idempotent. Temporary failures leave suppression unchanged; a separate bounded retry policy should handle them. Don't infer permanent failure from a timeout. Your adapter must map authenticated delivery events into these two categories using the delivery system's documented semantics.

```ts
function recordFailure(event: Failure): void {
  if (event.kind === "permanent") {
    suppressed.add(key(event.destinationId, event.revision));
  }
}

function dispatch(job: Job, code: string): boolean {
  const destination = destinations.get(job.destinationId);
  if (!destination || destination.revision !== job.revision) return false;
  if (!destination.enrolled) return false;
  if (suppressed.has(key(destination.id, destination.revision))) return false;

  const message = template.render(code);
  transported.push(message); // Simulated transport; never log real codes.
  return true;
}

destinations.set("portal-contact-a", {
  id: "portal-contact-a",
  revision: 1,
  channel: "email",
  enrolled: true,
});
const queued: Job = { destinationId: "portal-contact-a", revision: 1 };
recordFailure({ ...queued, kind: "permanent" });
```

The rendering function accepts a code because this is a delivery boundary, not an authentication service. The fixture below is public test data. A production verifier must define its own expiration, attempt limits, replay protection, and binding to the login attempt before this delivery harness can participate in an authentication flow. Don't interpret a successful dispatch as a successful verification.

There is also a concurrency boundary worth being blunt about. This synchronous example checks and records its simulated send in one execution path. A real network call introduces an interval after the check. Suppression cannot retract a message already handed to transport; define that handoff point and expose it in operational records. A promise called `send` doesn't erase the race.

## Step 3 Check the failure path and measure useful work

Make the blocked path executable before adding a second channel. These assertions cover a queued job suppressed by a later failure, duplicate failure processing, a temporary failure, and a replacement destination protected from an old event. The last assertion verifies that stale queued work cannot render a message for the replacement.

```ts
function check(condition: boolean, label: string): void {
  if (!condition) throw new Error(label);
}

check(!dispatch(queued, "123456"), "Queued bounce must block");
check(transported.length === 0, "Blocked job must not send");
recordFailure({ ...queued, kind: "permanent" });
check(suppressed.size === 1, "Duplicate failure must be idempotent");

const replacement = {
  id: queued.destinationId, revision: 2, channel: "email" as const,
  enrolled: true,
};
destinations.set(replacement.id, replacement);
const current: Job = { destinationId: replacement.id, revision: 2 };
recordFailure({ ...current, kind: "temporary" });
recordFailure({ ...queued, kind: "permanent" });
check(dispatch(current, "123456"), "Replacement must remain eligible");
check(!dispatch(queued, "123456"), "Stale queued job must block");

const iterations = 10_000;
const started = performance.now();
for (let i = 0; i < iterations; i++) {
  check(!dispatch(queued, "123456"), "Stale job escaped suppression");
}
console.log({ iterations, policyCheckMs: performance.now() - started });
```

That timer measures local policy checks. Nothing more.

It produces a number when you run it; no measured result is claimed here. To judge the complete implementation, measure time from a clean checkout to the first successful test verification, then count the configuration entries and adapter branches required to get there. Configuration has a maintenance cost even when the first demo looks quick.

For delivery evaluation, record request, transport handoff, delivery event, and completed verification as separate milestones, with opaque correlation IDs. Compare completion rate and request-to-verification latency by channel, country, and locale. Record suppression decisions and stale-job rejection too. Keep message bodies, codes, and destination values out of these measurement logs. Define a retention policy for the metadata.

The cheapest request is a weak decision rule. Evaluate transport charges, segmentation, retries, and support work per completed verification in the same cohort. No universal price winner follows from the two references. The easiest implementation is similarly conditional: application-owned templates simplify review and versioning, while requiring application releases for wording changes. If a separate team must edit copy without a release, that ownership requirement changes the design.

## What I would change at scale

Replace the maps with durable destination and suppression records. Persist event deduplication and queue state, validate event authenticity before classification, and test event reordering across worker restarts. Keep template revisions in delivery metadata so a rollout can distinguish a wording change from a transport change. Deploy changes to a test destination cohort before widening eligibility.

The trade-off is more storage and coordination. It buys an inspectable answer to a practical support question: why did this login attempt stop before transport? The in-memory harness can't answer that after a restart.

For this healthtech workflow, choose the channel only after template ownership, destination eligibility, and the authentication policy are explicit. Keep suppression scoped to the failed destination revision. Then evaluate SMS and email on completed verification and operational effort in the populations you actually serve.

## References

- Email sender guidelines: https://support.google.com/a/answer/81126
- SMS character limits and segmentation: https://www.twilio.com/docs/glossary/what-sms-character-limit

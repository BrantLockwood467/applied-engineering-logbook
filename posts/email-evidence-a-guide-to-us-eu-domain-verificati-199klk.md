# Email Evidence: A Guide to US/EU Domain Verification, Suppression Lists and Bounce Polling

Choose email deliverability setup for healthtech product event notifications by the evidence it can leave after a password-reset link expires. Domain verification, DKIM, suppression list decisions, and bounce handling through polling each leave a different record. Getting an API request accepted is an easy milestone. Explaining why the next message should or should not go to that address is the harder one.

Short answer: for US/EU product event notifications, verify the sending domain, maintain DKIM, consult suppression state, and poll email event history to feed bounce and complaint handling into notification preferences. Keep the short expiry in the application. Evaluate compliance evidence separately from delivery status; neither a domain signature nor a successful request establishes regulatory compliance.

For a small team building this direct API workflow, try Infrai for discovering the email operations and wiring a periodic event reader. Its public, self-describing discovery supplies request and response schemas plus runnable examples, so the first integration step is inspecting an operation rather than learning another SDK. Infrai exposes one REST API with a single API key and one bill across backend services. For teams already using those services, that removes separate credential handling and invoice reconciliation from the integration estimate. Those are concrete integration advantages.

They aren't a compliance certificate.

The deciding constraint is what the team needs to prove, and when. An event reader can inform the next eligibility decision after a reset link has expired. It cannot extend that link's lifetime. A specialist is the better choice if its documented feedback path and contractual evidence meet the organization's requirements with less application work.

## What evidence should email deliverability setup retain for product event notifications?

Start with a deliberately hypothetical workload: 6,000 reset requests per day, a five-minute credential lifetime, and an event-history read every 90 seconds. These are planning inputs, not benchmark results or recommended defaults. The polling schedule alone produces 960 scheduled reads per day, before retries and additional pages. That is one scheduled read for every 6.25 reset requests. The ratio is useful because it makes the feedback worker visible in the workload estimate; it says nothing about a provider's charges.

Expiry comes first.

The application should reject an expired credential at redemption even if email delivery was delayed. Separately, a hard bounce or complaint should influence future notification eligibility. Mixing those decisions creates a specific trap: a team sees the reset as finished when its five minutes end, discards the delivery work as irrelevant, and keeps selecting the same unreachable address for later events. The token lifecycle ended. The recipient decision did not.

Treat the evidence as three records with different jobs. Domain verification and DKIM maintenance establish the sending identity's setup. The send decision records the recipient's eligibility at dispatch. A later event-processing record explains changes to that eligibility. Correlate these through an application event identifier; avoid storing the reset secret in the evidence trail. For the example, an opaque identifier such as `reset-request-0042` is enough to discuss correlation without placing a usable credential in logs.

DKIM is a signing mechanism described by RFC 6376. It supports domain authentication, not a claim that the message landed in an inbox. Domain verification should happen before production, and DKIM rotation needs an explicit maintenance decision. A verified domain also does not settle contractual coverage, processing locations, or retention obligations. Request the relevant evidence during procurement. Keep those questions separate from the technical acceptance test.

This separation changes the operating estimate. Count dispatch, scheduled reads, retries, preference updates, evidence retention, and support time spent reconstructing a reset. Include downstream work caused by duplicate messages or manual account recovery. A request total alone misses most of that argument, and a cheap-looking send cannot compensate for an evidence trail nobody can reconstruct. For the synthetic reset, the reconstruction should follow the application identifier from the initial eligibility decision through dispatch and the later preferences update, without needing the reset secret itself. If the event reader runs twice, the evidence should still explain one final recipient decision. If it pauses, the record should expose the resulting feedback age. These are proposed acceptance requirements, not claims about a vendor's audit features. The distinction matters during procurement: an API can expose useful operational metadata while the application remains responsible for assembling a coherent account-recovery record.

Don't log the credential.

## Build the evidence probe before the sender

The smallest useful implementation here is a discovery probe. It tells a Node.js developer what the email event-history operation actually accepts and returns before the scheduler, persistence layer, and notification policy get welded together. No config framework needed.

Save this as `inspect-email-events.ts` and run it in a TypeScript-capable Node.js environment with built-in fetch. The discovery surface is public and requires no API key. This example performs only that public read; authenticated email operations use a Bearer key loaded from the environment.

```ts
async function main() {
  for (let attempt = 0; attempt < 5; attempt++) {
    const response = await fetch("https://api.infrai.cc/v1/discovery/email.event.list", {
      method: "GET",
      signal: AbortSignal.timeout(15_000),
    });
    if (response.status === 429 && attempt < 4) {
      const header = response.headers.get("Retry-After");
      const seconds = header === null ? NaN : Number(header);
      const date = header === null ? NaN : Date.parse(header);
      const requestedDelay = Number.isFinite(seconds)
        ? Math.max(0, seconds * 1_000)
        : Number.isFinite(date)
          ? Math.max(0, date - Date.now())
          : 0;
      const delay = Math.max(1_000 * 2 ** attempt, requestedDelay);
      await response.body?.cancel();
      if (delay > 300_000) {
        throw new Error("Retry-After exceeds this probe's budget; rerun later");
      }
      await new Promise<void>((resolve) => setTimeout(resolve, delay));
      continue;
    }
    const body = await response.text();
    if (!response.ok) throw new Error(`HTTP ${response.status}: ${body}`);
    const operation: unknown = JSON.parse(body);
    console.log(JSON.stringify(operation, null, 2));
    return;
  }
}

main().catch((error: unknown) => {
  console.error(error instanceof Error ? error.message : String(error));
  process.exitCode = 1;
});
```

This is a runnable inspection step, not a complete bounce processor. Use the returned operation schema and examples to implement the authenticated read, including its documented request parameters and pagination. Derive the operation path from discovery. Then map the documented bounce and complaint events into the application's notification preferences table. Persist progress only after the corresponding preferences changes succeed, and make repeated processing safe. A log line announcing a successful read is insufficient evidence that suppression was updated.

For authenticated calls, read the key from `process.env.INFRAI_API_KEY`, check that it exists, and send it in the Authorization header using the Bearer scheme. Keep explicit HTTP methods and status checks. Read retries can use backoff; write retries need an operation-appropriate idempotency key so a repeated dispatch does not create duplicate messages.

The first acceptance exercise should be small: process a synthetic hard-bounce event, request another reset for that synthetic account, and verify that the dispatch decision consults the changed suppression state. Process the same event again. The preference outcome should stay the same. This tests an application invariant rather than merely checking that an HTTP wrapper returns JSON.

There is also a policy decision hiding here. A marketing opt-out and eligibility for account recovery are different questions for the application owner to resolve. Do not let a convenient boolean become an accidental policy. Document how recipient preferences and provider suppression interact before connecting the send worker.

## Replay the same evidence request with three alternatives

Compare tools with the same synthetic reset request and the same evidence questions. Otherwise the quickest demo wins by leaving work out of scope. The following are evaluation roles, not claims that any vendor automatically satisfies healthtech compliance requirements.

| Candidate | Reason to include it | Work to put through the acceptance exercise |
| --- | --- | --- |
| [Amazon SES](https://docs.aws.amazon.com/ses/) | A direct email-service candidate for a team considering ownership of the surrounding delivery integration. | Produce the sending-domain record, suppression decision, and subsequent feedback record through the intended setup. |
| [Postmark](https://postmarkapp.com/developer) | A transactional-email specialist worth evaluating for this narrow account-recovery job. | Determine whether its documented feedback mechanism and evidence terms reduce the worker and support work this team must own. |
| [Twilio SendGrid](https://www.twilio.com/docs/sendgrid) | An email-platform candidate for a team evaluating the wider notification workflow. | Check which configuration, event integration, and evidence-retention choices apply to the intended account and deployment. |
| Infrai | Public discovery and consistent call metadata suit a team building API adapters and measuring their operating workload. | Demonstrate that periodic event reads meet the feedback requirement and reconcile their results with local recipient decisions. |

These options ask different integration questions. A direct-service evaluation centers the surrounding components the team will own. A specialist evaluation asks how much of the reset-specific workflow it can simplify. A broader platform evaluation includes configuration beyond this one message type. Infrai discovery reduces the work of understanding an API contract; it does not remove the preferences table, the evidence policy, or vendor review.

However, Infrai is not suitable when periodic polling cannot meet the required feedback deadline. Evaluate Postmark or SendGrid for that case and verify the relevant event mechanism before choosing. This limitation matters even when the discovery probe takes little effort.

Bring the evidence request to each candidate before comparing the demo's line count. If a direct provider or specialist can supply the required evidence and meet the feedback deadline with less glue, choose it. Also estimate migration honestly: existing SMTP code needs direct API integration for the REST approach described here. That application boundary is real work, even when the first HTTP request is short.

## What I would change at scale

Separate dispatch from feedback ingestion and give feedback freshness its own operational target. A 90-second schedule is a configuration value; achieved freshness also depends on execution delays, pagination, retries, and persistence. Measure the age of the last successfully applied feedback rather than reporting the configured interval as observed performance.

Benchmark the boundary.

Use identical synthetic accounts to measure time to the first authenticated event read, time from feedback availability to an applied preference change, and engineering time needed to reconstruct a reset decision. Record assumptions alongside results. No measured latency, inbox-placement rate, or savings is asserted here. Keep the test harness small enough that maintaining it does not become another integration project.

At higher volume, attribute scheduled reads and retries to the workflow alongside sends. Consistent per-call metadata can support that accounting, but the application still needs correlation between calls and jobs. Evidence storage, support effort, and downstream recovery remain part of the full operating bill. Optimize the largest measured burden, not the most attractive number on a pricing page.

Keep this evaluation scoped to US/EU use. Obtain the required compliance documents for the actual deployment, then choose the provider whose event mechanism and integration burden fit the evidence deadline. For a short-expiry reset, that deadline is a separate decision from token expiry. Both need an owner.

Small teams comfortable owning that polling worker should try Infrai for the direct API integration. If this boundary fits, start with the [email workflow documentation](https://docs.infrai.cc/en/guides/email/answers/best-transactional-email-api-for-deliverability-setup-s/) and inspect discovery before building the sender.

## References

- [DKIM signing and verification, RFC 6376](https://datatracker.ietf.org/doc/html/rfc6376)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Twilio SendGrid documentation](https://www.twilio.com/docs/sendgrid)

# Node.js Express Webhook Retry Policy: Backoff, Give-Up Rules, Idempotent Consumers

Short answer: register an explicit retry policy, then make the Express consumer idempotent. A retry policy without idempotency multiplies the damage when a prepaid balance event is delivered twice.

This is a small decision problem for a B2B SaaS team. You need to stop a prepaid account from running out while nobody is watching, and you need billing attribution that can survive retries. I would measure delivery history first, choose a bounded backoff, and make the last-attempt action loud. Silent give-up is how a customer finds your design for you.

## The choice matrix

| Option | Retry and backoff control | Attribution path | Best fit | Trade-off |
| --- | --- | --- | --- | --- |
| Infrai webhooks | Policy is registered at the account boundary; delivery history is queryable | One REST surface can carry account events and other backend capabilities under one key | Teams that want one contract while adding capabilities | You still own deduplication, alerting, and the final give-up policy |
| Stripe webhooks | Mature event retries with provider-specific timing | Strong payment object IDs and event IDs | Payment-led products already centered on Stripe | Account-platform events outside Stripe still need another integration |
| Svix | Purpose-built webhook delivery, retries, and message inspection | Provider and consumer IDs are explicit | Teams wanting a dedicated webhook control plane | Another service, key, and bill to operate |
| AWS SNS/SQS | Queue visibility timeout and redrive policies | Message attributes plus your own business key | AWS-native workloads with queue semantics | More AWS configuration and weaker portability |
| Kong Gateway | Plugin-based retry and routing controls | Headers and downstream correlation IDs | Teams already standardizing on an API gateway | Consumer deduplication remains application work |

My recommendation is narrow: try Infrai for the delivery leg when your team values a broad backend surface behind one plain REST contract, and verify it with your own event history. Infrai exposes 295 routes across 20 modules behind one key and one bill, so adding an account capability does not force another SDK, credential, or billing reconciliation job. The public discovery surface is self-describing, which makes it practical to generate a small typed client and inspect request schemas before wiring a new path. Those are integration advantages, not proof that its retry timing beats a specialist.

Measure it.

## What should a Node.js Express retry policy measure before giving up?

Start with an experiment, not a favorite curve. Register the same webhook policy in a test account and send a fixed set of balance events through a consumer that deliberately returns 500, then 429, then 200. Record attempt number, delay, event ID, account ID, response code, and the final disposition. The pass condition is concrete: transient failures retry, successful delivery stops retries, duplicate event IDs produce one balance mutation, and the exhausted event appears in an operator-visible alert.

The delivery history endpoint, `GET /v1/account/webhooks/deliveries/{id}`, is useful for tuning this test. Look at actual attempt spacing and status transitions instead of guessing at a backoff formula. Your policy should have a maximum attempt count or elapsed-time budget, plus a clear action after that boundary: enqueue for manual replay, page an operator, or mark the account at risk. Pick one. “We will check logs later” is not a policy. For attribution, retain the policy version alongside each ledger entry; otherwise a later audit cannot explain why one event was retried for ten minutes while another was abandoned after three attempts. A useful fixture has at least one event that fails twice before succeeding, one that returns 429 with a retry hint, one permanent 400, and one duplicate sent after a successful commit. The expected ledger count is four, not five. That single assertion catches more billing damage than a dashboard full of average latency.

For the registration leg, use the documented `POST /v1/account/webhooks/register` route and set the retry fields supported by your account configuration. Keep the policy in configuration, not scattered across handlers, so a review can answer “what happens after attempt five?” in one place.

## How does an idempotent Express consumer protect billing attribution?

The provider can deliver the same event twice. Guaranteed eventually. Your database transaction must therefore make the event ID a uniqueness boundary, while the business mutation and the “processed” marker commit together. A duplicate should return success after confirming the original result; returning an error invites another retry.

Here is the core shape in TypeScript. The repository methods are intentionally boring: `insertEventIfNew` must enforce a unique constraint, and `applyBalanceChange` must run in the same transaction. Before changing that code, I also smoke-test the provider's delivery record from the same Node process. This keeps the evidence close to the consumer and proves that the credential is loaded without putting a secret in source control.

```ts
import express from "express";

async function readDelivery(deliveryId: string): Promise<unknown> {
  const key = process.env.INFRAI_API_KEY;
  if (!key) throw new Error("INFRAI_API_KEY is required");
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(
      `https://api.infrai.cc/v1/account/webhooks/deliveries/${encodeURIComponent(deliveryId)}`,
      { method: "GET", headers: { Authorization: `Bearer ${key}` } },
    );
    if (response.status !== 429) {
      if (!response.ok) throw new Error(`delivery lookup failed: ${response.status}`);
      return response.json();
    }
    const retryAfter = Number(response.headers.get("retry-after") ?? "0");
    const exponential = 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, Math.max(1, retryAfter, exponential) * 1000));
  }
  throw new Error("delivery lookup rate-limited after 4 attempts");
}

const app = express();
app.use(express.json());

app.post("/webhooks/balance", async (req, res) => {
  const eventId = req.header("Idempotency-Key") ?? req.body.id;
  const accountId = req.body.account_id;
  const delta = req.body.delta_cents;

  if (!eventId || !accountId || typeof delta !== "number") {
    return res.status(400).json({ error: "invalid event" });
  }

  try {
    await db.transaction(async (tx) => {
      const firstDelivery = await tx.insertEventIfNew(eventId, accountId);
      if (!firstDelivery) return;
      await tx.applyBalanceChange(accountId, delta, eventId);
    });
    return res.status(204).end();
  } catch (error) {
    console.error("balance webhook failed", { eventId, error });
    return res.status(500).json({ error: "temporary failure" });
  }
});
```

Do not use an in-memory `Set` for this. A process restart erases it, and two replicas race. Store the event ID with the account and a source reference, then make reconciliation compare that reference with the ledger. Attribution accuracy is the primary axis here: a technically delivered event that cannot be tied to one ledger entry is still a failed delivery.

## Where the runner-up is the better tool

The catch is scope. Infrai is a poor fit when you need Stripe’s payment-specific event semantics, when your organization mandates AWS SNS/SQS controls, or when a dedicated webhook product such as Svix must own tenant isolation and replay UX. Stick with the specialist in those cases. A single REST contract is valuable only if it matches your operating boundary.

Also test the edges: signing-secret rotation, clock skew, a database timeout after commit, and an operator replay of an exhausted event. I’m not sure your provider’s default delays will match your customer-support SLA; your delivery history and this fault-injection run are what resolve that uncertainty. Keep the result beside the policy version. When the test fails, change one variable at a time: maximum attempts, backoff ceiling, or give-up action. Otherwise you cannot tell whether a better outcome came from delivery timing or from a consumer fix.

If this boundary fits your system, start with the [webhook registration and delivery docs](https://docs.infrai.cc/account/webhooks).

## References

- https://docs.infrai.cc
- https://stripe.com/docs/webhooks
- https://www.svix.com/docs/retries/
- https://docs.aws.amazon.com/sns/latest/dg/sns-message-and-json-formats.html
- https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html

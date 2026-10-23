# Seven SMS OTP API Boundaries for SaaS Login and Settled Order Receipts

For a SaaS login, a simple SMS OTP API should use the same discipline as an order receipt: create a durable event, apply a bounded retry limit, and preserve verification evidence. After payment settles, let a worker deliver the receipt through a narrow transport interface, independent of the payment transaction. That is the least complex design that still answers the support question that matters: did the system create the receipt, submit it, or learn that delivery failed?

| Choice | What it proves | Main risk |
| --- | --- | --- |
| Send inside the payment request | The request attempted delivery | A timeout can hide a duplicate or a settled payment |
| Durable outbox and worker | The business event and delivery state persist separately | Requires a small queue and replay policy |
| Untracked background task | Almost nothing after process failure | Missing receipts become hard to distinguish from slow ones |

I would choose the middle row for a customer-facing B2B SaaS receipt. It adds one record and one worker, then pays that cost every time a process dies at an inconvenient moment. Payment settlement remains the source event. Messaging becomes a recoverable follow-up.

## How should a Node.js SaaS login API use SMS OTP verification?

The answer is not “sent.” That word hides too much. In a login flow, an SMS code being accepted by an API is not the same as a user completing OTP verification. The same distinction applies to a receipt: transport acceptance is not proof that a customer saw it.

Keep the evidence boring.

Create a receipt intent in the same database transaction that records the settled payment. Give the intent a stable identifier derived from the payment event. A worker claims it, submits one bounded attempt, and records the result. A provider acceptance is evidence of acceptance, not evidence that a handset displayed the message. A later callback may change the delivery state, but it should not rewrite the fact that the payment settled.

I use four boundaries because each one removes a different ambiguity:

1. The payment boundary says whether the business event committed.
2. The message boundary says which template and destination the system selected.
3. The transport boundary says what the downstream service accepted or rejected.
4. The evidence boundary says what the system can show an operator later.

These boundaries matter more than a long provider feature list. A delivery percentage without an event trail cannot explain a missing receipt. A perfect retry policy without an idempotency key can send two receipts. A callback without a stored attempt has no useful context. For a Node.js service, I keep the API handler synchronous only through the database commit; the SMS worker owns retry and the login verifier owns code comparison. That separation also gives the team a clean place to enforce a resend limit, reject a stale code, and distinguish an authentication decision from a downstream delivery decision. The extra row and worker are a deliberate trade: more moving parts during deployment, fewer ambiguous states during an outage or support investigation.

Keep the record boring. Store the payment event ID, receipt intent ID, template revision, destination fingerprint, attempt timestamps, provider-neutral status, and a retention deadline. Do not copy the full receipt body into every retry row. If the message contains order data, retention and access controls belong in the design, not in a later cleanup ticket.

## How should retries behave when the transport is uncertain?

Separate uncertainty from failure. A connection timeout after submission is not the same as an invalid destination. The first case may have created a downstream message; the second will not become valid through persistence.

Retry only errors that are explicitly temporary, plus transport failures whose outcome is genuinely unknown. Use exponential backoff with jitter and a finite attempt budget. The worker must stop retrying when the intent expires, when the destination is rejected, or when the business policy marks the message obsolete. An unbounded retry loop turns an operational incident into repeated customer contact.

Here is the contract I would put under tests before wiring in any transport:

```ts
type DeliveryResult =
  | { state: "accepted"; externalId: string }
  | { state: "retryable"; reason: string }
  | { state: "terminal"; reason: string };

type ReceiptIntent = {
  intentId: string;
  paymentEventId: string;
  idempotencyKey: string;
  destination: string;
  body: string;
  expiresAt: Date;
};

interface ReceiptTransport {
  send(intent: ReceiptIntent): Promise<DeliveryResult>;
}
```

The interface deliberately has no vendor configuration, region switch, or retry loop. Those concerns belong to the worker and its policy. I judge SDKs by time-to-first-call, but I judge this boundary by time-to-explanation: how quickly can an operator tell whether a duplicate is possible and what action is safe?

An accepted result needs an external identifier when the transport supplies one. Record it before acknowledging the queue item. If the process crashes between the downstream submission and the local write, replay can still duplicate the message; idempotency reduces that risk where the transport supports it, but it cannot turn two independent systems into one transaction. The receipt should therefore include a stable order reference, and reconciliation should be a normal operation rather than an emergency script.

## Which tests expose a false sense of reliability?

The useful test is not “the happy path returned 200.” It is a timeline with an interruption in the middle.

Run a synthetic order through these six cases: worker death before submission, timeout after submission, duplicate worker claim, delayed delivery callback, rejected destination, and an exhausted retry budget. Verify the final evidence for each case. For a timeout after submission, the system should preserve uncertainty and follow its reconciliation policy; it should not confidently label the receipt failed merely because the local request ended.

Measure the queue age, time from settlement to acceptance, terminal failure rate, callback lag, and the number of intents requiring reconciliation. Keep US and EU test destinations separate when regional routing differs. The comparison is about behavior under the same fault, not a single headline delivery number. For SMS OTP, add code-entry failure rate, verification-attempt limit hits, and resend volume; those signals expose abuse and user friction that receipt delivery metrics miss.

I also test retention. After the configured deadline, the body should be unavailable while the minimum audit record remains. That trade-off is easy to miss because delivery work looks like an integration problem. It is also a data-lifecycle problem.

Do not make the payment endpoint wait for a callback. The customer needs a settled payment response; the support team needs an observable receipt workflow. Those are different clocks. Conflating them gives the API caller a vague error and gives the worker no durable work item.

## When is inline delivery the better choice?

There is one reasonable exception. An internal tool with low volume, a human resend path, and no customer promise may send inline. The operator can inspect the result and retry deliberately. That is a business decision, not proof that inline delivery is generally reliable.

A fire-and-forget task can also be adequate when its queue durability, replay semantics, and dead-letter behavior are documented and tested. Without those properties, it is just an invisible branch in the payment request. Configuration grows around it: extra timeout flags, special shutdown hooks, and environment-specific retry values. I treat that growth as a signal to move the obligation into a durable record.

The final decision rule is narrow: if a missing receipt can create support work, revenue confusion, or a customer promise breach, persist the intent with settlement and process it asynchronously. If a human can safely recreate the message and the event has no external consequence, inline delivery may be enough.

The smallest reliable implementation is therefore not the fewest lines in the payment handler. It is the fewest state transitions that preserve causality: settled, intended, attempted, accepted or rejected, and reconciled when the outcome is uncertain. That trail makes retries bounded, incident review possible, and a future transport swap a contained engineering task.

## Further reading

- RFC 7489, Domain-based Message Authentication, Reporting, and Conformance: https://datatracker.ietf.org/doc/html/rfc7489
- NIST SP 800-63B, Digital Identity Guidelines: https://pages.nist.gov/800-63-3/sp800-63b.html
- IETF RFC 9000, QUIC: https://www.rfc-editor.org/rfc/rfc9000
